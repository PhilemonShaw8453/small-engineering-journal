# SaaS SMS Alerts API: Prefer Direct Resets, Reserve Bulk for Incidents

Choose direct dispatch for a property portal's short-lived password-reset SMS when reducing integration effort is the priority. Reserve bulk fanout for a separate SaaS incident-alert workflow. Both can share a transport adapter, but a reset request needs its own expiry decision before every send attempt.

Short answer: compare SMS APIs by the work required to enforce that boundary, test retries, and interpret delivery events. A cheapest-provider or no-monthly-minimum verdict for US and EU traffic needs current SMS contract and route evidence. The sources in References do not establish those terms, so this note makes no price ranking.

The useful comparison here is direct dispatch versus bulk fanout. For a tenant trying to regain access, choose the smaller request path while the work is one recipient and one expiring credential. When the job becomes notifying many residents about an incident, evaluate a separate fanout path with its own scheduling and recipient rules.

That's a different workload.

## What must happen before a reset SMS leaves the queue?

Use this proposed flow: the property portal accepts a reset request, an authentication component issues the credential and its expiry, and a worker receives an opaque request ID with the intended recipient. The worker checks the deadline immediately before handing the message to a transport adapter. Subsequent transport events update an attempt record; credential redemption remains the authentication component's responsibility.

Keep expiry authoritative.

For this example, choose 300 seconds (five minutes) as a fixture lifetime, not a security standard or a claim about any provider. A retry at second 299 still deserves a deadline check; a retry at second 300 must stop. If a bulk queue delays work until after that boundary, accepting the batch is no help to the tenant. The integration question becomes concrete: where can the application enforce this rule, and how much custom state does that require?

## Start with an executable expiry check

This Python example is a local evaluation harness. It sends no network traffic, creates no credentials, and uses an opaque fixture handle in place of message content. The fake clock makes the boundary reproducible. The transport interface is deliberately small so the same cases can later run against an adapter without changing the decision rule.

```python
from dataclasses import dataclass
from typing import Callable, Protocol


@dataclass(frozen=True)
class ResetJob:
    request_id: str
    recipient_ref: str
    credential_ref: str
    expires_at: int


class Transport(Protocol):
    def submit(self, job: ResetJob, *, attempt_key: str) -> str:
        ...


class RecordingTransport:
    def __init__(self) -> None:
        self.calls: list[tuple[str, str]] = []

    def submit(self, job: ResetJob, *, attempt_key: str) -> str:
        self.calls.append((job.request_id, attempt_key))
        return f"receipt-{len(self.calls)}"


class ResetDispatcher:
    def __init__(
        self, transport: Transport, now: Callable[[], int]
    ) -> None:
        self.transport = transport
        self.now = now
        self.receipts: dict[str, str] = {}

    def dispatch(self, job: ResetJob) -> tuple[str, str | None]:
        if self.now() >= job.expires_at:
            return "expired", None
        if job.request_id in self.receipts:
            return "already_submitted", self.receipts[job.request_id]
        receipt = self.transport.submit(job, attempt_key=job.request_id)
        self.receipts[job.request_id] = receipt
        return "submitted", receipt


def evaluate() -> None:
    clock = [0]
    transport = RecordingTransport()
    dispatcher = ResetDispatcher(transport, lambda: clock[0])
    job = ResetJob(
        request_id="property-reset-001",
        recipient_ref="tenant-fixture-17",
        credential_ref="credential-fixture-001",
        expires_at=300,
    )

    clock[0] = 299
    assert dispatcher.dispatch(job) == ("submitted", "receipt-1")
    assert dispatcher.dispatch(job) == ("already_submitted", "receipt-1")
    clock[0] = 300
    assert dispatcher.dispatch(job) == ("expired", None)
    assert len(transport.calls) == 1

    clock[0] = 301
    delayed_transport = RecordingTransport()
    delayed = ResetDispatcher(delayed_transport, lambda: clock[0])
    assert delayed.dispatch(job) == ("expired", None)
    assert delayed_transport.calls == []
    print("Expiry and repeated-dispatch checks passed.")


if __name__ == "__main__":
    evaluate()
```

The four checks cover submission before expiry, repeated dispatch, the exact deadline, and a job first processed too late. They demonstrate the local decision only. A submission receipt means the fake adapter returned a receipt; it makes no assertion about handset delivery. That distinction should survive the notebook-to-prod transition. The interesting pitfall lies between submission and recording its result. Suppose a real adapter accepts the request, then the worker exits before storing the receipt. This in-memory dictionary cannot resolve what happened. Before deployment, require a durable attempt record and define the adapter's behavior for an ambiguous outcome. Verify any claimed deduplication support against its documented scope and retention period. Passing an attempt key to an interface doesn't establish that a downstream service honors it. If a second worker takes over, its acceptance test must distinguish a known receipt from an unresolved attempt; blindly creating another request ID would discard the very correlation needed to investigate the first submission.

That's the next test.

## Should a bulk SMS alerts API handle SaaS incidents and password resets together?

Direct dispatch wins this proposed design because each reset request already supplies the unit of work: one recipient, one credential reference, one deadline. Its limitation is that it does not provide incident-wide recipient expansion or campaign scheduling. A bulk abstraction becomes attractive when a single event must create many recipient jobs, but for this reset flow it adds a second grouping concept that the authentication decision does not need. This is a conditional architectural choice, not a measured throughput claim.

For an API comparison, ask each candidate to demonstrate the same acceptance cases. Can the adapter expose an ambiguous submission result without labeling it delivered? Can the application preserve the deadline through retries? How are transport events matched to a durable attempt? How much recipient, sender, and route configuration must the team implement for its intended US and EU destinations? Record unanswered questions as evaluation gaps. Don't fill them with assumptions about a brand.

Integration effort should include the operational work after the first successful request. Estimate time for sender configuration, event authentication, retry policy, privacy review, and on-call diagnosis using the team's own trial. Those are proposed evaluation categories, not assertions that every service has identical requirements. A small adapter is exciting precisely because the acceptance harness can stay stable while those answers differ.

Bulk incident alerts still deserve their own test. Use a separate fixture population, simulate an incident burst, and measure queue age against the reset workflow's chosen deadline if infrastructure is shared. Avoid treating a successful bulk submission as evidence that an individual tenant can redeem a reset credential on time.

Use this proposed decision record to keep those outcomes separate:

| Observed application state | Reset action | What remains unknown |
| --- | --- | --- |
| Deadline reached before submission | Stop this reset job | Whether the tenant will request a new reset |
| Adapter returned a receipt | Record the receipt | Handset delivery and credential redemption |
| Submission outcome ambiguous | Preserve the attempt for resolution | Whether transport accepted it |

## Keep the reset path deterministic and the comparison honest

An AI-enabled property portal can still use a fixed reset template and deterministic dispatch logic. In this design, keep generation out of credential handling. That leaves no model call in the proposed send path and makes the evaluation independent of prompt variation. Put any optional drafting experiment on incident copy behind a separate review boundary, with synthetic recipients and no live credentials.

Prompt-cost awareness belongs in that boundary too: measure an optional model step separately from transport usage, so a long prompt cannot disappear inside a communications estimate. No token price is needed to decide whether the reset workflow needs generation at all.

Here, it doesn't.

The email sources provide a narrower kind of evidence. The email API introduction in References does not establish SMS route coverage, delivery timing, or commercial terms. The FTC guide explains how an email's primary purpose affects treatment of mixed commercial and transactional content. Use that as a reason to review an email fallback's content; it does not establish permission to send SMS in either region. For this design, keep the reset message focused on access recovery and obtain the applicable SMS requirements separately.

A no-monthly-minimum claim also needs a scope. Ask for written terms covering the intended account, sender arrangements, destinations, and expected traffic before recording that claim in a comparison. Treat the answer as contract evidence. The current material supports no numerical price verdict.

## Move the harness into operations

Before enabling live reset messages, make the expiry and repeated-dispatch checks part of the adapter acceptance run. Add cases for timeout after submission, worker restart, concurrent processing of the same request, and repeated or out-of-order transport events. The sample has no cross-process coordination; the production design must specify who may claim an attempt and what happens when that claim expires. Also test credential redemption separately, including an expired credential and the chosen policy for a newer reset request.

For observability, use opaque request and attempt identifiers, queue age, submission outcome, and expired-before-submit counts. Keep live reset credentials out of logs, evaluation artifacts, and prompts. Review the proposed telemetry with the team that owns authentication and decide which delivery events can safely change user-visible status. A transport event must not extend the credential's expiry in this design.

Then stage the rollout with synthetic recipients, confirm how operators investigate an ambiguous send, and assign ownership for template and sender changes. Measure integration effort across that whole exercise. The recommendation remains direct dispatch for short-lived property resets, with bulk fanout evaluated separately for incident alerts. Choose the shared infrastructure only after its queue and retry behavior pass the same deadline checks.

## References

- Email API introduction; background on an email integration, not evidence of SMS capabilities or pricing: https://resend.com/docs/introduction
- FTC guidance on commercial and transactional email, including mixed-content primary-purpose rules: https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business
