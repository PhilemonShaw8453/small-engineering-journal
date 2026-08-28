# Bounce Suppression for Email and SMS Event Notifications: Webhook vs Polling Comparison

Short answer: for event notifications, accept app alerts through a durable send queue, use webhooks when bounce suppression must update quickly, and keep polling as a reconciliation path; choose an email and SMS API only after measuring the integration work for both paths.

The operational constraint is the feedback loop, not the first API call. A provider may accept an email or SMS before the final delivery state exists. If the application treats that acceptance as delivery, an invalid recipient stays eligible and the next alert repeats the mistake. The simplest design tried in a notebook often stops at `send()`. The production design needs a second plane for delivery events, plus a suppression decision that survives retries and deploys.

This note compares webhook and polling architectures rather than ranking vendors. Twilio, SendGrid, Customer.io, Courier, Knock, and other candidates can all go through the same evaluation harness. The useful question is how much application-owned machinery each candidate leaves behind.

## Should event notifications use webhook or polling for email and SMS app alerts?

Use a webhook as the primary signal when a hard email bounce or an invalid SMS destination must suppress later notifications promptly. The callback should do very little: authenticate the event using the provider's documented mechanism, persist the raw envelope, deduplicate it, enqueue normalization, and acknowledge it. Parsing, suppression, fallback, and product analytics belong after that durable boundary.

Keep a poller when the provider exposes event lookup and delayed convergence is acceptable. Polling is also useful as reconciliation even with webhooks, because the evaluation question is not "did our endpoint return success?" but "can our local ledger converge with the provider's event history after a missed or duplicated callback?" The exact retention window and replay contract vary by provider. I'm not sure a candidate is safe to reconcile until its documentation or a controlled test establishes both.

Polling alone can be the smaller integration when alerts are low urgency and the product tolerates the polling interval. There is no public callback service to operate, no inbound authentication path, and fewer moving parts during a notebook-to-prod transition. The catch is direct: every minute between polls is another minute in which a newly invalid address may remain sendable. Polling also creates repeated read traffic, so the worker must stop querying terminal messages and must honor the provider's documented rate-limit behavior.

Webhooks invert that cost. They reduce detection delay without periodic reads, but add an internet-facing endpoint, secret rotation, replay handling, queue backpressure, and an explicit policy for out-of-order events. Don't update a recipient row directly from the request handler. One duplicated or late event should not be able to roll a terminal suppression backward.

There is no magic transport.

The balanced pattern is webhook-first with scheduled reconciliation for high-consequence delivery state, and polling-only for workflows where slower feedback is an honest product choice. This is an integration-effort decision: count the callback service, queue, datastore, scheduler, observability, and test fixtures. A feature checkbox hides most of that bill.

## Model bounces and suppression before comparing providers

Start with an application-owned notification ledger. Each row needs a stable application event ID, recipient identity, channel, provider message ID once known, current normalized state, and timestamps for the last observed event. Keep the raw provider event separately from the normalized state. That separation lets an adapter change without rewriting business history, and it gives an eval harness something deterministic to replay.

Suppression is a policy result, not a string copied from a callback. A hard email bounce and a transient delivery delay should not produce the same action. Nor should an email outcome silently block a phone number. Define the scope of a suppression key: recipient, destination, channel, tenant, and reason. Then define which normalized events may create it, which later events may supersede it, and whether any removal requires an explicit user action. Those choices belong in code review because they affect both deliverability and whether a person receives an important alert.

A focused adapter can stay small. The example below assumes an authenticated ingress layer has already produced a dictionary. It makes no claim about any provider's payload names; provider-specific code maps into this internal event before calling the reducer.

```python
from dataclasses import dataclass
from datetime import datetime
from enum import StrEnum


class DeliveryState(StrEnum):
    ACCEPTED = "accepted"
    DELIVERED = "delivered"
    TRANSIENT_FAILURE = "transient_failure"
    PERMANENT_FAILURE = "permanent_failure"


@dataclass(frozen=True)
class DeliveryEvent:
    event_id: str
    message_id: str
    destination_key: str
    channel: str
    state: DeliveryState
    observed_at: datetime


@dataclass(frozen=True)
class SuppressionDecision:
    suppress: bool
    reason: str | None


def decide_suppression(event: DeliveryEvent) -> SuppressionDecision:
    if event.state is DeliveryState.PERMANENT_FAILURE:
        return SuppressionDecision(
            suppress=True,
            reason=f"permanent_{event.channel}_failure",
        )
    return SuppressionDecision(suppress=False, reason=None)
```

This reducer is intentionally boring. It doesn't send a fallback channel, mutate a user profile, or infer that `accepted` means `delivered`. A worker can apply it inside the same database transaction that records the event ID, making duplicate delivery harmless. Another worker can consult the suppression table before creating the next outbound message.

The longer failure case deserves attention. Imagine that a callback is persisted, the process exits before normalization, and the provider retries it. The unique event ID prevents a second raw record, while the pending record remains available to the queue consumer. Now imagine a delivered event and a permanent-failure event arrive out of order. The reducer alone cannot choose precedence; the state machine needs documented provider semantics and a local monotonic rule. Build fixtures for both sequences. This is where an eval-driven approach earns its keep: the same fixture suite runs against every adapter, and a provider swap changes parsing code rather than suppression policy.

SMS adds a separate payload test. Message length depends on character encoding and segmentation; Twilio's character-limit documentation explains the GSM-7 and UCS-2 distinction. Render the final personalized message in the test harness, then inspect its encoding and segment behavior. Testing the template before variable substitution misses the characters that can change the result.

## Compare integration effort with a replayable eval harness

A comparison spreadsheet is useful only if its cells correspond to executable checks. Start with one fixture corpus: accepted, delivered, permanent failure, transient failure, duplicate event, unknown event, and deliberately reversed order. Add a recipient who is already suppressed and verify that no new send intent is created. The corpus should exercise email and SMS independently.

Then score architecture, not brand familiarity.

| Decision area | Webhook evidence to collect | Polling evidence to collect |
|---|---|---|
| Authentication | Documented verification inputs and secret-rotation path | Documented API authentication and credential scope |
| Idempotency | Stable event identity under retry | Stable event or message identity across pages |
| Recovery | Replay or redelivery behavior | Event retention, cursor behavior, and terminal states |
| Ordering | Meaning of timestamps and late events | Ordering across pages and overlapping poll runs |
| Operations | Acknowledgement deadline and retry policy | Rate limits and a safe polling interval |
| Data boundary | Payload fields and regional handling | Stored event fields and regional handling |

Run the adapter against captured sandbox fixtures, then measure what the application actually cares about: time from provider event to local suppression, duplicate-event rate after deduplication, oldest unreconciled message, callback queue depth, poll duration, and sends prevented by active suppression. Token cost matters in an AI application too, but it belongs in the rendered-content evaluation rather than the transport choice: prompt-generated alerts should be length-bounded before they reach either channel, and the exact rendered output should be stored with the send intent for reproducibility.

This also exposes organizational cost. A team that already operates authenticated webhook ingress and durable queues may add a provider adapter cheaply. A small service with a scheduler and no public ingress may find polling materially easier. Conversely, a product promise of near-immediate channel fallback makes polling-only unsuitable, however pleasant the first integration looks. Stick with polling-only when delayed suppression is acceptable; require webhook delivery when reaction time is part of the user-facing behavior.

Vendor categories complicate the shortlist. A channel API and a notification-orchestration platform may both send a message, yet leave different work in the application. The comparison should record who owns preferences, fallback rules, templates, suppression, and event normalization. Product names from an initial search query are candidates for the harness, not conclusions: Twilio, SendGrid, Customer.io, Courier, and Knock should neither receive credit nor lose points until their current documentation and sandbox behavior answer the same test cases. Resend's introduction is likewise useful for understanding its documented API surface, but an introduction page cannot establish replay and suppression semantics by itself.

## What should be measured before copying this architecture?

First, set the product's maximum acceptable suppression delay. That single number separates a legitimate polling-only design from one that merely postpones webhook work. Then establish the expected alert volume and burst shape, because average traffic won't reveal callback queue pressure or an overlapping poll run.

Measure recovery, too. Disable the consumer in a controlled environment, accumulate events, restore it, and verify convergence from durable inputs. Replay duplicates. Reverse events. Rotate an authentication secret using the documented procedure. A test passes only when the local ledger and suppression state settle correctly without sending another alert to the invalid destination.

Finally, price the engineering surface in things the team will maintain: public endpoints, credentials, queue capacity, scheduled jobs, state storage, dashboards, and adapter fixtures. Provider fees can change and aren't enough to decide the architecture. The durable choice is the one whose failure modes the team can rehearse and whose delivery state can be explained from stored evidence.

Keep the send path dull. Spend the design effort on feedback.

## References

- https://resend.com/docs/introduction
- https://www.twilio.com/docs/glossary/what-sms-character-limit
