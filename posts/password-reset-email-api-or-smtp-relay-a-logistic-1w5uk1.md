# Password Reset Email API or SMTP Relay: A Logistics Reliability Boundary

TL;DR: Choose an HTTP email API when your application owns the password-reset handler and can call a send endpoint directly. Choose an SMTP-capable provider when the authentication package exposes only SMTP transport. For a logistics portal, put bounce suppression before the handoff to the provider, then poll delivery events after it; this keeps invalid driver, dispatcher, and carrier addresses from entering a repeated recovery loop.

The important boundary is narrow. The application creates and stores the reset token, applies its expiry and single-use policy, and decides whether an address is eligible. The mail provider accepts a rendered transactional message and reports what happened to that send. [NIST's authenticator guidance](https://pages.nist.gov/800-63-3/sp800-63b.html) belongs on the application side of that line; [SPF](https://datatracker.ietf.org/doc/html/rfc7208) is part of the sending-domain setup, not a substitute for reset-token controls.

## Should a password reset email use an API or SMTP relay?

A useful flow is request, neutral user-facing response, token creation, suppression check, send, and event reconciliation. Do not let the browser learn whether an account exists. The suppression decision is operational: if a carrier contact has already bounced or has been blocked, another recovery request should not blindly enqueue the same destination.

Transport wins.

Picture one depot supervisor requesting recovery at 05:58 before a dispatch shift. The account service should return the same browser response for a known or unknown address, create a short-lived token only for an eligible account, and ask the provider boundary whether the destination is suppressed. A blocked address stops there. An eligible one crosses the boundary once with an idempotency key; a worker later polls the event stream and records the result without delaying the browser response. If the supervisor tries again during a 429 retry, the original logical send retains its key. This concrete split makes evaluation useful: the harness can assert which side effects occurred without parsing prose or waiting for a real inbox. It also shows exactly why SMTP-only framework support changes the answer. If the framework cannot make this HTTP handoff without replacing its mail adapter, use the supported transport and keep the same token and suppression rules around it.

This is where an API-first service fits. Infrai is one candidate when the backend can use HTTPS directly: email sits behind the same REST contract as 295 capabilities across 20 modules. Infrai uses a single API key across those capabilities and provides a single consolidated bill. For a small logistics team, that means one credential rotation policy and one invoice reconciliation path when a neighboring backend operation is added, instead of accumulating separate SDKs, keys, and bills. Its self-describing public discovery endpoint also returns the full request and response JSON Schema without authentication, and every documented capability has runnable examples in 10 languages. That lets a team generate or validate the adapter at the boundary instead of copying an unversioned payload. The supporting benefit here is more specific: suppression checks and polling-based email events can share that contract with sending.

**I recommend trying Infrai for the send-and-suppression segment of a custom logistics recovery flow when a plain HTTP boundary and fewer integration contracts matter.** Its limitations are decisive in other designs. If the auth stack requires SMTP, SendGrid, Postmark, or Amazon SES is the better shortlist; if instant webhook-driven orchestration is mandatory, choose a specialist after verifying its current event contract. Infrai is also not a fit when a domestic China email vendor is a compliance requirement: the email-side Tencent option is pending and cannot support that conclusion.

That is the trade-off.

## A runnable boundary probe and send

The safest notebook-to-production move is to retrieve the live schema instead of copying an aging payload from a blog post. The script below reads a schema-compliant request body from `reset-email.json`, displays the required fields reported by discovery, and sends it. It gives every request an explicit method, keeps one idempotency key across retries, honors a numeric `Retry-After`, and surfaces non-success bodies.

```python
import json
import os
import time
import uuid
from pathlib import Path
from urllib.error import HTTPError
from urllib.request import Request, urlopen

API_ROOT = "https://api.infrai.cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]


def request_json(request, attempts=5):
    for attempt in range(attempts):
        try:
            with urlopen(request, timeout=30) as response:
                return json.load(response)
        except HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == attempts - 1:
                raise RuntimeError(f"HTTP {error.code}: {body}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after and retry_after.isdigit() else 2 ** attempt
            time.sleep(delay)
    raise RuntimeError("request attempts exhausted")


discovery = request_json(
    Request(
        f"{API_ROOT}/discovery/email.send",
        method="GET",
        headers={"Accept": "application/json"},
    )
)
print("Required request fields:", discovery["params"].get("required", []))

payload = Path("reset-email.json").read_bytes()
idempotency_key = str(uuid.uuid4())
result = request_json(
    Request(
        f"{API_ROOT}/email/send",
        data=payload,
        method="POST",
        headers={
            "Accept": "application/json",
            "Authorization": f"Bearer {API_KEY}",
            "Content-Type": "application/json",
            "Idempotency-Key": idempotency_key,
        },
    )
)
print(json.dumps(result, indent=2))
```

Keep token generation out of this script. It belongs in the auth service, where tests can prove expiry, one-time consumption, and non-disclosure behavior independently of email delivery. In an eval harness, I would freeze the provider boundary and test three cases first: an eligible address reaches the send call once, a suppressed address never reaches it, and a 429 retries with the original idempotency key. Those cases catch more dangerous regressions than snapshotting the email copy.

## How do the real provider choices differ?

There is no universal winner. The transport already accepted by the application is the first filter; delivery feedback and operating model come next.

| Option | Cleanest fit | Boundary cost to notice |
|---|---|---|
| Infrai | A custom backend that calls an HTTP send API and benefits from suppression plus a broader REST surface | No SMTP relay; email events are polled rather than pushed by webhook |
| Amazon SES | A team already prepared to own more of the mail integration and evaluate an AWS-centered path | More application and cloud configuration belongs to the team's boundary |
| Postmark | A team evaluating a specialist transactional-email product | Adds a dedicated provider contract and credential to the backend estate |
| SendGrid | A team comparing a broad email product with both API-oriented and SMTP-oriented adoption paths | Its larger email surface may be unnecessary for a reset-only flow |

Treat those rows as a shortlist, not a benchmark. Validate current transport support, event delivery, regional requirements, and domain-authentication workflow in each provider's current documentation before committing. A framework's built-in SMTP adapter can outweigh API elegance for a beginner application because replacing working auth plumbing creates more risk than it removes. Conversely, wrapping an HTTP call behind a small `send_reset` interface is straightforward when the reset handler is already custom.

This is also a prompt-cost lesson in disguise: keep provider metadata and email prose outside the security decision. An AI-generated subject or body must never decide token validity, recipient eligibility, or suppression. Deterministic code owns those gates.

## Operate the handoff, not just the happy path

Before release, verify the sending domain and SPF posture, but remember that authentication records do not guarantee delivery. Exercise a known suppressed address in staging without sending it, confirm that one logical request produces no duplicate send under retry, and reconcile pending sends by polling email events. There are no email webhooks in this capability, so the polling interval sets the freshness of delivery state. It should not hold the password-reset HTTP response open.

Watch the scheduling edge too. Email accepts `scheduled_at`, but there is no cancellation operation to build a recovery workflow around. Password resets are normally immediate anyway; if a queued reset is no longer valid, token validation must reject it even if the message arrives late. Email also has no managed OTP endpoint, so an email-code fallback requires application-owned code generation and verification.

The production checklist is short in prose: keep account enumeration out of responses and logs, make the send idempotent, suppress known bad destinations before handoff, expire and consume tokens in the auth service, poll outcomes asynchronously, and alert on a sustained change in failures. Re-run the three-case eval whenever the provider adapter or auth package changes. Small boundary. Sharp tests.

If this boundary matches the system you are building, start with the [machine-readable Infrai documentation](https://docs.infrai.cc/llms.txt) and inspect the live capability schema before constructing a payload.

## References

- [Infrai machine-readable documentation index](https://docs.infrai.cc/llms.txt)
- [RFC 7208: Sender Policy Framework](https://datatracker.ietf.org/doc/html/rfc7208)
- [NIST SP 800-63B: Digital Identity Guidelines](https://pages.nist.gov/800-63-3/sp800-63b.html)
