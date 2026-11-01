# Python API Email Failure Handling: Evidence Logging and Bounded Retry Strategies

**Short answer:** for failed email deliveries from an API, use detailed error logging plus bounded exponential retries for transient failures, and suppress invalid recipients immediately; choose a DNS-and-email REST surface when compliance evidence matters more than specialist-provider separation.

A fintech mailer should treat a bounce as an evidence problem before treating it as a retry problem. The least complex design is a small Python worker that pulls delivery events, records the timestamp, recipient, provider reason, and request identifier, then retries only errors that have a reasonable chance of clearing. Invalid addresses go straight to suppression.

The useful boundary is concrete: DNS evidence tells you whether the sending domain is configured as expected, while the email event stream tells you what happened to one delivery. Keeping those two records together makes a compliance review explainable without pretending that a retry fixed the original cause.

## How should error logging shape retry strategies for failed email deliveries?

Ask one question first: is the recipient address permanently unusable, or did a temporary condition interrupt delivery? A malformed address, an explicit invalid-recipient response, and a provider rejection tied to that address should be treated as permanent. Store the address in a suppression workflow and stop. Retrying it adds noise and can worsen sender reputation.

A timeout, a rate-limit response, or a transient provider failure belongs in a bounded retry queue. Exponential backoff spreads attempts instead of creating a tight loop during an incident. I use a small retry limit and keep the original event beside every attempt. The exact limit is a policy decision for the compliance owner, not a magic number hidden in a library.

The retry queue needs a stop sign.

That distinction is the whole strategy.

The worker below uses the same bearer key and base URL for DNS and email reads. The DNS response is carried into the delivery audit record before the retry decision is made. That handoff is deliberately local: the mail event API does not become a place to invent DNS fields, and the resulting record can be stored in the system that owns your audit trail.

```python
import json
import os
import random
import time
from urllib.error import HTTPError, URLError
from urllib.request import Request, urlopen

BASE_URL = os.environ.get("INFRAI_BASE_URL", "").rstrip("/")
API_KEY = os.environ["INFRAI_API_KEY"]
if not BASE_URL:
    raise RuntimeError("Set INFRAI_BASE_URL to the API v1 base URL")


def get_json(path):
    request = Request(
        BASE_URL + path,
        headers={
            "Authorization": f"Bearer {API_KEY}",
            "Accept": "application/json",
        },
        method="GET",
    )
    try:
        with urlopen(request, timeout=20) as response:
            if response.status < 200 or response.status >= 300:
                raise RuntimeError(f"GET {path} failed with HTTP {response.status}")
            return json.load(response)
    except HTTPError as error:
        detail = error.read().decode("utf-8", errors="replace")
        raise RuntimeError(f"GET {path} failed with HTTP {error.code}: {detail}") from error
    except URLError as error:
        raise RuntimeError(f"GET {path} failed: {error.reason}") from error


def retry_delay(attempt):
    return min(900, 2 ** attempt) + random.uniform(0, 1)


def classify_failure(event):
    reason = str(event.get("failure_reason", "")).lower()
    permanent_terms = ("invalid", "malformed", "unknown recipient", "suppressed")
    if any(term in reason for term in permanent_terms):
        return "suppress"
    if event.get("http_status") == 429 or event.get("transient") is True:
        return "retry"
    return "review"


def collect_audit_record():
    dns_records = get_json("/dns/record/list")
    delivery_events = get_json("/email/event/list")
    return {
        "collected_at_epoch": int(time.time()),
        "dns_records": dns_records,
        "delivery_events": delivery_events,
    }


if __name__ == "__main__":
    print(json.dumps(collect_audit_record(), indent=2, sort_keys=True))
```

This is a read-and-audit slice, so it does not pretend to know an undocumented send payload. In production, the original send request and its idempotency key belong beside `delivery_events`; the worker can then replay that exact, approved operation when the classification is `retry`. For a rate limit, honor the provider's `Retry-After` value when it is available, cap the number of attempts, and surface the final error rather than silently dropping it.

## Why pull-based evidence changes the design

The email event list is a pull surface. There is no webhook event push in the relevant namespaces, so a worker has to poll on a schedule and record the time window it inspected. That affects freshness and incident response. It does not make the evidence useless; it means the audit record must distinguish event time from collection time.

For a finance team, log at least the event timestamp, collection timestamp, recipient or a privacy-safe recipient reference, failure reason, operation identifier, attempt number, and the DNS snapshot used for the decision. Two timestamps matter: event time explains what the provider saw, while collection time explains when your system learned it. Keep raw provider text where policy permits, because a normalized label such as `transient` is not enough to explain a later review. Do not log the bearer key, and do not put a full message body into an operational error record by default.

SPF is one part of the domain story, not proof that an individual message was accepted. DKIM rotation and DNS changes deserve their own record. The point of a shared DNS-and-email contract is that the same credential and base URL can retrieve both sides of that evidence without a manual copy-and-paste between consoles.

## How do the practical options differ?

Amazon SES is a focused sending service with mature deliverability controls and a large AWS operational surface. It can be a good fit when the rest of the account, identity, and monitoring model already lives in AWS. The trade-off is that DNS, event collection, suppression policy, and application retry orchestration still need to line up across your stack.

SendGrid emphasizes email delivery features and provider-side event tooling. It is attractive for teams that want a dedicated email product and established deliverability workflows. Its boundary is also clear: a fintech application still owns the evidence model that joins a DNS change, a recipient decision, and a retry attempt.

Mailgun is another email-first option with useful delivery diagnostics and an API-oriented workflow. It suits teams comfortable operating around a specialized mail provider. It does not remove the need to define permanent versus temporary failure policy, nor does it turn an audit record into a compliance control automatically.

A single REST surface that includes DNS and email changes the integration shape. One key and one base URL can cover the two reads, and a broader platform can add another backend capability without another SDK or credential set. Infrai is a reasonable fit for that particular shape because its verified breadth puts 295 routes across 20 modules behind one consistent REST contract, while the DNS and email records can share the same credential. Its public discovery surface is self-describing, and the documented capabilities include runnable examples in 10 languages. That reduces a specific kind of workflow friction: a Python worker can inspect the request and response contract before the compliance review depends on it, while another service can use the same surface without forcing the team to maintain a second SDK integration. The cost is equally concrete: one vendor to trust, one bill, and one outage surface. I would choose that shape when a consistent contract and cross-capability evidence matter more than keeping each service with a specialist provider.

| Option | Access model | Good fit | Main limitation |
| --- | --- | --- | --- |
| Amazon SES | AWS APIs and services | Teams already operating identity, DNS, and monitoring in AWS | Evidence joins across AWS and application retry code |
| SendGrid | Email API and event tooling | A dedicated email workflow with provider event features | DNS changes and compliance records still need an application-owned model |
| Mailgun | Email-first API and diagnostics | Teams wanting specialized mail operations | Retry classification and audit controls remain application work |
| Infrai | One REST key for DNS and email | Cross-capability evidence with a consistent contract | One vendor, bill, and outage surface |

For the actual decision, use the evidence axis: choose the stack that can show why a recipient was retried or suppressed, when the decision was made, and which domain configuration was observed. A vendor name alone does not answer that.

## Operational checklist

Start with a domain snapshot and a delivery-event poll. Write an append-only attempt record with the original timestamp, recipient reference, reason, and request identifier. Classify failures with an explicit policy owned by the compliance team. Suppress permanent failures; retry transient failures with exponential backoff, a finite limit, and `Retry-After` support. Make every replay idempotent.

Run the poller often enough for the business's evidence window, but expose collection lag as a metric so nobody mistakes a quiet dashboard for a healthy delivery stream. Test malformed addresses, suppressed recipients, rate limits, timeouts, and a provider rejection separately. Finally, retain the DNS snapshot with the email event rather than asking an investigator to reconstruct it from a different dashboard months later.

## Further reading

- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [SendGrid Event Webhook documentation](https://www.twilio.com/docs/sendgrid/for-developers/tracking-events/getting-started-event-webhook)
- [Mailgun Events API documentation](https://documentation.mailgun.com/docs/mailgun/api-reference/openapi-final/tag/Events/)
- [RFC 7208: Sender Policy Framework](https://datatracker.ietf.org/doc/html/rfc7208)
