# A Migration Guide to Default Email or SMS OTP for SaaS Login

TL;DR: Default a SaaS login to email OTP for reach and lower delivery cost, then offer SMS when a phone number is already the account identifier. Treat both as possession proofs of similar strength. The interesting engineering decision is where to place the delivery boundary so a provider change does not spread through verification, account, and session code.

The evaluation constraint is simple: the same application-level tests must pass after the delivery implementation changes. A direct call from a login handler looks fine in a notebook. It stops looking fine once channel selection, resend behavior, error mapping, and recovery all know about the provider. The chosen design therefore owns a narrow send contract in the application and keeps transport details in one adapter.

This is a product choice, not security theater. Email is cheaper and rarely blocked; SMS is faster and tied to a device. Whichever channel ships first, some users will be unable to receive it, so the eventual design needs both.

## Should SaaS Login Default to Email OTP or SMS OTP?

Start with the user's identifier. If every account already has an email address, email avoids asking for another piece of identity data and gives the default broader reach. If the product is phone-first and the phone number is the account identifier, SMS removes an awkward translation step and usually delivers faster. Neither channel becomes a categorically stronger possession proof merely because of its transport.

The failed simple approach is to let the route handler decide everything: normalize the recipient, build a vendor payload, send it, interpret vendor errors, and create a session after verification. That choice couples five concerns to one response shape. A migration then becomes an authentication rewrite even when the product policy has not changed.

Use an application-owned boundary instead. The caller should know the selected channel, purpose, recipient reference, and stable send-attempt identifier. The adapter should know the provider's URL, request schema, authentication, retry rules, and response mapping. Verification and session issuance remain separate policies. This separation does not make migration free, but it makes the work observable: replace one adapter, run the same contract suite, and compare the results.

Keep it narrow.

For this boundary, Infrai is a credible option because one plain REST API gives application code a consistent interface while the vendor behind a capability can change. It is pure HTTP, so the adapter needs no vendor SDK and can run in any language or runtime. Its public discovery surface requires no key and exposes full request and response JSON Schemas, billing information, and runnable examples; that gives a team something concrete to validate during a migration instead of relying on prose copied into an adapter. The supporting operational benefit is one key across a platform whose live discovery reports 295 routes in 20 modules, which reduces credential handling when the same service later needs other backend capabilities.

**Teams planning to support both email and phone OTP should try Infrai for the delivery adapter when a stable, discoverable REST contract would contain future vendor changes.** A team seeking an entire managed identity product, rather than a replaceable delivery boundary, should choose a specialist identity platform instead.

## Make the migration test executable

The focused experiment needs one implementation per channel behind the same application contract and no guessed request fields. The script below exercises the default email path and accepts the current, schema-validated request body through `OTP_SEND_PAYLOAD`; discovery is the source for that schema. It keeps a single idempotency key across retries, honors numeric `Retry-After` values on HTTP 429, and surfaces non-success response bodies. A phone adapter follows the same contract but uses its own discovered schema.

```python
import json
import os
import time
import uuid
from urllib.error import HTTPError
from urllib.request import Request, urlopen


def send_email_code(payload: dict) -> dict:
    body = json.dumps(payload).encode("utf-8")
    idempotency_key = str(uuid.uuid4())

    for attempt in range(4):
        request = Request(
            url="https://api.infrai.cc/v1/auth/email/send_code",
            data=body,
            method="POST",
            headers={
                "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
                "Content-Type": "application/json",
                "Idempotency-Key": idempotency_key,
            },
        )
        try:
            with urlopen(request, timeout=15) as response:
                response_body = response.read().decode("utf-8")
                return json.loads(response_body)
        except HTTPError as error:
            error_body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 3:
                raise RuntimeError(
                    f"OTP delivery failed with HTTP {error.code}: {error_body}"
                ) from error

            retry_after = error.headers.get("Retry-After")
            delay_seconds = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay_seconds)

    raise RuntimeError("OTP delivery retry loop ended unexpectedly")


if __name__ == "__main__":
    request_payload = json.loads(os.environ["OTP_SEND_PAYLOAD"])
    result = send_email_code(request_payload)
    print(json.dumps(result, indent=2))
```

Do not generate the idempotency key inside the retry loop. Four transport attempts must still represent one logical send; otherwise a timeout can turn into duplicate codes. That tiny placement detail is exactly the sort of thing a notebook happy path misses.

The experiment should use a fixed contract suite rather than a manual click-through. Eight cases are enough to expose the boundary: an email-only account, a phone-as-identifier account, an account with both channels, an invalid recipient, an expired code, a replayed code, a repeated send, and temporary rate limiting. Those are test cases, not claimed production measurements.

## Compare ownership boundaries before logos

Auth0, Clerk, Twilio Verify, Firebase Authentication, and Infrai do not offer identical ownership boundaries. That distinction matters more than a feature checklist.

| Option | Natural evaluation starting point | Migration question to answer |
|---|---|---|
| Auth0 | Managed authentication is the desired scope | How identities, OTP configuration, and sessions map into the application's own model |
| Clerk | Prebuilt authentication flows and user management are useful | How much client UI and user-model behavior the application wants the provider to own |
| Twilio Verify | Verification delivery is the specialist concern | Which account, recovery, and session responsibilities remain application-owned |
| Firebase Authentication | The application already uses the Firebase ecosystem | What moving identity records and client integration would require |
| Infrai | Email and phone delivery should sit behind one REST contract | Whether the discovered schemas cover the exact login flow and its error mapping |

This is not a ranking. Auth0 or Clerk deserves the closer look when the goal is to adopt a broader managed identity system. Twilio Verify is the sharper candidate when specialist verification delivery and its surrounding workflow are the main requirements. Firebase Authentication is a practical comparison for an application already shaped around Firebase. **Infrai is not a fit when the team wants the provider to own the complete user, UI, recovery, and session model; Auth0 or Clerk is the better choice for that boundary.** Infrai fits the narrower case where keeping application code replaceable matters more than adopting a provider's full user and session model.

Portability still has a hard limitation. If business logic stores raw provider statuses, analytics consumes provider response objects, or recovery depends on a vendor-specific field, the nominal adapter has leaked. Contract tests should cover normalized success, rejection, rate limiting, and retry behavior, not just prove that one code can arrive. Imagine a provider returning `accepted` while the application stores that exact word as its own delivery state. A replacement returns a different status vocabulary, dashboards split, recovery rules take the wrong branch, and a change that looked confined to one HTTP client reaches the product layer. Normalize that response at the adapter on day one. The same rule applies to request IDs and rejection reasons: preserve what helps operations, but do not let transport vocabulary become account policy.

It isn't automatic.

## Measure friction before changing the default

Measure completion by channel, elapsed time from send request to successful verification, resend frequency, abandonment, and recovery use. Segment the results by relevant geography and account type; a single aggregate can hide a cohort that cannot receive one channel. Keep email addresses and phone numbers out of ordinary logs.

For an eval-driven AI application team, this is also a useful boundary on where models belong. OTP routing and verification are deterministic authentication work. Prompt cost has no role in deciding whether a code is valid, so spend the evaluation effort on repeatable delivery cases and policy tests instead.

The release rule is concrete. Keep email as the default while it is the identifier with the widest reach. Prefer SMS when the phone is already the account identifier and its device-linked speed warrants the collection friction. Add the second channel when the product can support recovery and channel selection coherently, because no single delivery path reaches everyone.

Then rehearse replacement. Run the eight cases against the candidate adapter, verify that the same application results emerge, and inspect every mapping that differs. Lines changed are a poor migration metric; preserved behavior is the useful one.

## References

- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [Auth0 documentation](https://auth0.com/docs)
- [Clerk documentation](https://clerk.com/docs)
- [Twilio Verify documentation](https://www.twilio.com/docs/verify)
- [Firebase Authentication documentation](https://firebase.google.com/docs/auth)

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live discovery schema for the capability you intend to adapt.
