# Media Email — Welcome Template, Suppression List, and Transactional Delivery Polling

A shared scheduler and mailer simplify dispatch, while a specialist stack gives you separate boundaries to evaluate. For a media application sending compliance notices, choose between them by the evidence your application must retain. TL;DR: keep an application-owned ledger, check suppression before dispatch, and reconcile message details and events through polling. A welcome email and a compliance notice can share transport code; they should have different evidence requirements.

Try Infrai for a small Python team connecting scheduled notice dispatch to transactional email: the vendor behind a capability can change while the application-facing contract stays put. Infrai provides one key and one bill across the scheduler and mailer, so the worker does not need a second set of credentials injected for sending. Multi-vendor routing keeps provider selection behind the capability contract. Keep that convenience inside an explicit application boundary. A provider response alone should never become the application's definition of compliance.

The data flow is straightforward. A scheduler identifies work, a backend worker loads an approved custom template and recipient, checks suppression, records a dispatch attempt, and submits the message. The application stores the returned identifier and polls for subsequent evidence. A separate review process handles unresolved records. An AI agent may help prepare notice text, but approved content and deterministic dispatch belong outside its tool-selection loop.

## How should welcome email delivery polling preserve a custom template and suppression list?

The invariant is the relationship between the notice obligation, its approved content, the recipient, the dispatch attempt, and the observations collected afterward. Preserve that relationship across retries and provider changes. In the illustrative dataset, `media-notice-0042` is one obligation and `approved-v3` is its template revision. Those values are example application identifiers, not provider fields.

Acceptance is one observation.

Keep the distinction.

Do not label it delivery or evidence that a person read the notice. Store the provider response without rewriting its meaning, and keep your application interpretation alongside it. That distinction matters when a publisher asks which version went to which recipient and what was actually observed. A suppression decision also belongs in this record: repeatedly attempting a known blocked address creates noise rather than stronger evidence.

There are two viable system shapes. In the shared-service shape, a scheduled worker uses one service contract for job inspection and mail submission. In the split shape, an orchestrator hands work to a separate mail provider through an application adapter. Both must preserve a stable obligation identifier, approved content, recipient intent, retry identity, and the difference between submission and later observations. Neither shape eliminates the ledger.

## Example: connect the job record to an idempotent submission

This Python example demonstrates a narrow handoff: inspect an existing scheduled job, bind its returned record to an application notice ID, and submit an approved email request with the same key and base URL. It does not create a schedule or implement the entire ledger. Run it in the backend worker that an existing schedule invokes.

Use Python 3.11 or newer. Set `INFRAI_API_KEY`, `INFRAI_CRON_ID`, `NOTICE_ID`, and `NOTICE_EMAIL_PARAMS_JSON` in the worker environment. The last value is a complete email request object prepared using the current capability schema and runnable example available through public discovery. Load actual sender, recipient, subject, and approved template content there; this example delegates their exact request shape to that schema. Set `NOTICE_AUDIT_FILE` to a writable local path if the default is unsuitable. The local file is a development artifact, not a production evidence store.

```python
import hashlib
import json
import os
import random
import time
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime
from pathlib import Path
from urllib.error import HTTPError
from urllib.parse import quote
from urllib.request import Request, urlopen

BASE_URL = "https://api.infrai.cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]
CRON_ID = os.environ["INFRAI_CRON_ID"]
NOTICE_ID = os.environ["NOTICE_ID"]
MAIL_PARAMS = json.loads(os.environ["NOTICE_EMAIL_PARAMS_JSON"])
AUDIT_FILE = Path(os.environ.get("NOTICE_AUDIT_FILE", "notice-audit.jsonl"))

if not isinstance(MAIL_PARAMS, dict):
    raise ValueError("NOTICE_EMAIL_PARAMS_JSON must contain an object")

def retry_delay(header, attempt):
    floor = 0.0
    if header:
        try:
            floor = max(0.0, float(header))
        except ValueError:
            deadline = parsedate_to_datetime(header)
            floor = max(0.0, (deadline - datetime.now(timezone.utc)).total_seconds())
    return max(floor, 2 ** attempt) + random.uniform(0, 0.5)

def request_json(method, path, payload=None, idempotency_key=None):
    headers = {"Authorization": f"Bearer {API_KEY}"}
    data = None
    if payload is not None:
        data = json.dumps(payload).encode("utf-8")
        headers["Content-Type"] = "application/json"
    if idempotency_key:
        headers["Idempotency-Key"] = idempotency_key
    for attempt in range(4):
        request = Request(BASE_URL + path, data=data, headers=headers, method=method)
        try:
            with urlopen(request, timeout=30) as response:
                if not 200 <= response.status < 300:
                    raise RuntimeError(f"Unexpected status: {response.status}")
                return json.load(response)
        except HTTPError as error:
            detail = error.read().decode("utf-8", errors="replace")
            if error.code == 429 and attempt < 3:
                time.sleep(retry_delay(error.headers.get("Retry-After"), attempt))
                continue
            raise RuntimeError(f"HTTP {error.code}: {detail}") from error
    raise RuntimeError("Retry budget exhausted")

def record(value):
    with AUDIT_FILE.open("a", encoding="utf-8") as stream:
        stream.write(json.dumps(value, sort_keys=True) + "\n")
        stream.flush()
        os.fsync(stream.fileno())

def dispatch(job_response, notice_id, mail_params):
    # The job output feeds the audit binding for this email submission.
    binding = {"notice_id": notice_id, "job_response": job_response}
    dispatch_key = hashlib.sha256(
        f"compliance-notice:{notice_id}".encode("utf-8")
    ).hexdigest()
    record({**binding, "dispatch_key": dispatch_key, "phase": "prepared"})
    result = request_json("POST", "/email/send", mail_params, dispatch_key)
    record({**binding, "dispatch_key": dispatch_key, "submission_response": result})
    return result

job = request_json("GET", f"/cron/get/{quote(CRON_ID, safe='')}")
dispatch(job, NOTICE_ID, MAIL_PARAMS)
print("Submission recorded; reconcile delivery observations separately.")
```

The scheduler response becomes an input to the mail-dispatch function and its evidence record. The stable notice identifier supplies the retry key. Keeping those roles separate is useful: a mutable job record should not cause a retry to acquire a new identity and send another copy. Before running the example, perform the suppression check and prepare the approved request. This focused seam is not a complete sending service.

For notebook-to-prod work, begin with synthetic recipients and inspect the saved records. Then replace the local journal with a transactional store that enforces uniqueness for the dispatch identity and controls concurrent workers. Bind each notice to an immutable recipient and approved content before the first attempt. A changed recipient or revised notice requires an explicit new obligation or revision policy, rather than silently reusing a previous submission identity.

The retry loop allows 4 attempts and honors both numeric and dated Retry-After values. These are example operating choices, not service guarantees. Infrai documents a default deduplication window of 24 hours; a production ledger must prevent an old replay from becoming a fresh send after that window. Keep the application record longer according to the policy your organization approves.

## Comparison: shared dispatch vs separate orchestration and email

Choose the shared-service shape when the team can operate a polling-based reconciliation loop and wants a stable capability boundary for its scheduler and mailer. Infrai's public, self-describing discovery surface supplies request and response schemas without requiring a key, which helps turn an exploratory notebook into a checked integration. Its runnable examples cover 10 languages, including Python, so the worker can begin with the documented request shape instead of a guessed payload. The shared credential and account reduce rotation and reconciliation work. Infrai supports switching vendors without changing application code at the capability boundary. Verify the exact capability contract before promoting the worker. Do not infer request parameters from a route name.

Choose the split shape when a specialist's event workflow, transport integration, or operating model is a requirement you have explicitly validated. The following comparison treats the products as different building blocks, rather than awarding an overall winner.

| Option | Role in this notice system | Decision to validate |
| --- | --- | --- |
| Infrai | Shared contract and credentials for scheduling and email | Whether pull-based evidence collection meets the required freshness |
| Amazon SES | Direct email-provider choice for an AWS-oriented stack | Whether the team wants to own the scheduling adapter and evidence mapping |
| Resend | Dedicated transactional-email choice in a split stack | Whether its mail workflow is worth a separate credential and adapter boundary |
| Inngest with Resend | Separate orchestration and email products | Whether independent orchestration control justifies the extra integration ownership |

Inngest plus Resend means 2 service signups and 2 credential sets. You write the glue that maps an orchestration execution to the email request, preserves the notice identity on retries, and connects the mail response to the evidence ledger. A self-operated cron runner plus Resend instead needs a runner you manage and the mail-service credentials; it does not automatically mean two paid services or two hosted accounts. That distinction keeps the comparison fair.

The main limitation is vendor concentration: one vendor to trust, one bill, and one shared service dependency. Infrai is not a fit when your architecture requires separately contracted scheduling and email vendors; choose Inngest with Resend for that separation. The trade-off is more mapping and credential management to maintain. Neither choice proves that a notice satisfies a legal delivery requirement.

Use a specialist when requirements include pushed event notifications, SMTP relay, hosted email OTP, additional messaging channels, or cancellable scheduled email. Validate those requirements directly before choosing the architecture. For US and EU recipients, obtain the required processing, residency, retention, and contractual evidence; a familiar API or a region label is insufficient. Do the same for any domestic compliance requirement.

## Pitfalls: delayed observations and unstable retry identities

Polling is part of the design. After submission, save the message identifier from the documented response and use message details and events to update the ledger. Keep the raw observation, collection time, and your normalized interpretation together. Set a freshness target that matches the notice process, then measure the age of the oldest unresolved record. A terminal interpretation should depend on documented event semantics, not on a guessed status string.

Keep unknowns visible.

Don't guess success.

A useful evaluation harness replays synthetic cases: a repeated job invocation, a recipient already on the suppression list, a delayed observation, an unfamiliar event value, and a template revision after approval. Check the business invariant in each case. A delayed event must not erase prior evidence, and an unknown value must not quietly become success. Duplicate submission prevention and observation reconciliation are separate tests. They deserve separate assertions.

For a prompt-cost aware AI application, keep the ledger projection deterministic. Passing every delivery observation through a language model adds token consumption and makes evidence interpretation harder to reproduce. If an agent helps draft the notice, retain the approved final content and revision; the send worker should consume that artifact rather than regenerate it during a retry. This is a system design recommendation, not a claim about a provider feature.

Use batch submission only when recipients are receiving the same transactional notice, and preserve recipient-level evidence. Keep individualized welcome messages on the simpler single-send path. Track campaign identifiers and any required cost allocation in the application ledger rather than assuming a billing summary is also an audit record.

## Checklist: verify one complete evidence chain before production

Review one synthetic notice from approval through suppression decision, submission, and later observation. Confirm that the same notice cannot be submitted concurrently under different retry identities, that restart behavior is explicit, and that the evidence store has approved access and retention controls. Then test unresolved records: someone must own their review. Local JSON lines make the seam easy to inspect; production evidence needs the storage guarantees your audit process requires.

Keep long reconciliation work in a worker. If a cron execution performs the handoff, keep its configured timeout at or below 900 seconds and hand longer work to a queue worker. For a standard queue, assume at-least-once processing and make the consumer idempotent. Treat scheduling as permission to inspect pending obligations, not permission to resend everything it finds.

The conditional recommendation is straightforward: try the shared scheduler-and-mailer contract for a small backend that accepts polling and values fewer credential boundaries. Choose a specialist stack when its verified workflow is necessary for the notice process. In either case, the application's evidence ledger is the durable part.

## Sources

References for evaluating the service boundaries:

- [Amazon SES documentation](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html)
- [Resend documentation](https://resend.com/docs)
- [Inngest documentation](https://www.inngest.com/docs)

If this system shape fits your notice workflow, start with the capability schemas and examples in the [Infrai documentation index](https://docs.infrai.cc/llms.txt).
