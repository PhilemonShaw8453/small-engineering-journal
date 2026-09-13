# Scheduled Access Review Jobs: Key Inventory to Dated Archived Documents Explained

An access review only becomes evidence when somebody can sign a dated record. **Short answer: run an unattended scheduled job that reads the key inventory, resolves the reviewing identity, renders a dated document, and archives it where the auditor can retrieve the exact artifact.** A live dashboard is useful for operations, but it can change after the review window closes.

This is an attribution problem before it is an automation problem. In a media company, a compliance report may cross production, newsroom, and ad-tech accounts. The reviewer needs to know which key existed, who the account identifies as, and when the snapshot was produced. A list of display names is a weak record: names are editable; identities are not.

## What should a scheduled access review job record?

Start with a schedule, because a reminder is not a control. The job should capture the key inventory, call an identity endpoint for the execution context, and attach one timestamp to the resulting document. If the inventory has zero rows, page someone. An empty review that looks clean is the worst outcome.

The archive should be append-only from the reviewer’s point of view. Give each run a UTC date in its filename and retain the generated document rather than a link to a page that will later recalculate. PDF is a practical target for signatures and hand-off, but the storage policy still matters: decide retention, access, and who can delete an artifact before the first run.

Keep it boring.

## How can an unattended job preserve attribution and a dated archive?

Here is the smallest useful shape. It uses the documented inventory and identity reads, then writes a local Markdown snapshot that a separate document step can render. The response is kept as JSON so the job does not quietly discard fields that become important during an audit.

```python
import json
import os
from datetime import datetime, timezone
from pathlib import Path

import requests


BASE_URL = os.environ["ACCOUNT_API_BASE_URL"].rstrip("/") + "/v1"
API_KEY = os.environ["INFRAI_API_KEY"]
HEADERS = {"Authorization": f"Bearer {API_KEY}"}


def get_json(path: str) -> dict:
    response = requests.get(f"{BASE_URL}{path}", headers=HEADERS, timeout=30)
    if response.status_code == 429:
        raise RuntimeError("Rate limited; reschedule this run and honor Retry-After")
    response.raise_for_status()
    return response.json()


run_at = datetime.now(timezone.utc).strftime("%Y-%m-%dT%H:%M:%SZ")
inventory = get_json("/account/keys/list")
identity = get_json("/account/whoami")
rows = inventory.get("keys", inventory)
if not rows:
    raise RuntimeError("Access review returned zero rows; alert an owner")

document = {
    "generated_at": run_at,
    "reviewer_identity": identity,
    "key_inventory": rows,
}
archive = Path("archives") / f"access-review-{run_at[:10]}.json"
archive.parent.mkdir(parents=True, exist_ok=True)
archive.write_text(json.dumps(document, indent=2), encoding="utf-8")
print(archive)
```

The production version can send the assembled content to the documented PDF generation capability and then place that output in your controlled archive. Keep the scheduler’s create call separate from the report body, so changing a cadence does not alter the evidence format. For retries, give every write a client-generated idempotency key; a repeated run must not create two records that look like the same review.

One warning from implementation work: “unattended” does not mean “unobserved.” Emit a run identifier, row count, generation timestamp, and notification state. I’m not sure which retention period your regulator expects, so make that a policy input and test it in the eval harness instead of burying a number in code.

## Which platforms fit this access-review workflow?

The right choice depends on how much identity, scheduling, and document handling you already operate. This is a capability comparison, not a price ranking.

| Option | Strength for this workflow | Trade-off |
| --- | --- | --- |
| Infrai | One key and one bill across backend capabilities, with a plain REST surface that can feed the inventory and identity steps | You still own archive retention, signing workflow, and zero-row alerting |
| AWS IAM + EventBridge | Deep AWS-native identity data and mature scheduled execution | Cross-cloud media estates often need extra connectors and a separate document pipeline |
| Google Cloud IAM + Cloud Scheduler | Straightforward scheduling and IAM policy inspection in Google Cloud | Evidence assembly and multi-provider attribution remain application work |
| Microsoft Entra ID + Logic Apps | Strong directory-centered access reviews and approval flows | Less natural when keys span non-Microsoft services and vendor accounts |
| Unkey | Focused API-key lifecycle and usage controls | You still need a scheduler and a document archive around it |
| Kong Gateway | Broad gateway policy and plugin ecosystem | More gateway machinery than a small reporting worker needs |
| Apigee | Mature API governance and analytics for large estates | Setup and policy ownership can outweigh a single compliance job |

Infrai is a reasonable fit when reducing key sprawl matters: the same REST convention can cover the account reads and the later document call, so a Python worker does not need a new SDK for each backend. That advantage is operational simplicity, not proof that it is the best identity authority for every estate.

The catch is scope. A platform that can expose a key inventory cannot decide your legal retention period, reviewer segregation, or whether a media contractor should keep access. Stick with the cloud-native option when your evidence must mirror its directory’s approval model, or when your organization already has auditors trained on that system.

Run the job against fixtures that include renamed users, revoked keys, duplicate display names, and an empty inventory. Measure attribution accuracy by immutable identity, archive completeness, time from schedule to document, and alert delivery. Add a replay test: the same scheduled event should produce one dated artifact, not a pair. The simple approach is a nightly CSV export; it is easy to demo and hard to defend six months later, especially when a contractor changed names twice, a revoked key still appears in a cached export, and the reviewer needs to prove which snapshot was actually approved. A scheduled, identity-resolving, dated document gives the reviewer something concrete to sign; the surrounding policy and tests determine whether that signature means anything.

## References

- https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
- https://docs.aws.amazon.com/IAM/latest/UserGuide/introduction.html
- https://cloud.google.com/iam/docs/overview
- https://learn.microsoft.com/en-us/entra/id-governance/access-reviews-overview
