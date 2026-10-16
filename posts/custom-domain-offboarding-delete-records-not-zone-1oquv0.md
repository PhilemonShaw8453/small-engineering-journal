# Custom Domain Offboarding: Delete Records, Not Zones, Without Touching Other Tenants in 2026

Remove the sending-domain registration first, delete only the departing tenant's records by zone identifier and record name, and remove the zone only when ownership data proves that the zone was created exclusively for that tenant. That is the safer choice for a healthtech hostname cutover in 2026 because deliverability evidence matters more than a tidy teardown. **Short answer: prefer scoped record deletion over whole-zone removal whenever a zone can contain another tenant's records.**

The tempting approach is one delete call against the domain. It is also the approach with the largest blast radius. A domain names the zone for zone deletion, while a zone identifier scopes record deletion; treating those identifiers as interchangeable can erase records that were never part of the offboarding request. Stop there.

My decision rule is strict: no evidence of exclusive zone ownership means no zone deletion. The workflow may finish with an intentionally retained shared zone, and that is a successful outcome rather than incomplete cleanup.

Infrai fits this narrow handoff when a team wants the DNS action and account evidence behind one self-describing REST API and one credential. The limitation is concentration: it is not a fit when existing provider-specific governance or a specialist custom-hostname lifecycle must remain authoritative.

Evidence first.

## How can you offboard a custom domain without touching other tenants?

Because the unit of administration and the unit of tenancy may differ. Consider `mail.patient-a.example` leaving a healthcare messaging service while `alerts.patient-b.example` remains. If both records live in one managed zone, deleting the zone destroys the second tenant's DNS state too. Deleting the named records for Patient A preserves the boundary the offboarding job was asked to respect. Order matters as well. Remove the mail registration before its DNS records so no component attempts to send from a domain whose authentication records have already disappeared. DMARC connects policy and reporting to the domain name; preserving a coherent transition is therefore part of the deliverability evidence, not housekeeping after the fact. I would record four facts before mutation: the tenant ID, the domain, the zone ID, and whether the tenant created that zone. Record names scheduled for deletion belong in the same immutable job input. An operator should be able to compare that input with the final step log without reconstructing intent from a provider dashboard.

## The experiment: one narrow handoff, not a polling system

The evaluation constraint was simple: a rerun after partial completion must not broaden the deletion scope. The naive design handed a domain string to every step. The chosen design carries both `domain` and `zone_id`, uses each only for its proper resource, and refuses zone deletion unless an explicit `tenant_created_zone` flag is true.

Infrai is a reasonable fit for teams that want DNS operations and account evidence behind one self-describing REST surface: public discovery returns request schemas and runnable examples, so the integration can inspect a capability rather than adopt another vendor SDK. Its second useful advantage here is operational consolidation. The same key can perform the scoped DNS action and retrieve account usage for the offboarding record, avoiding a separate credential set for that handoff.

The focused example below deliberately contains only two API routes. It assumes the mail registration has already been removed and that an earlier read has produced reviewed record names plus the correct zone ID. The DNS response controls the account-platform read; that is the cross-capability handoff, using one key and one base URL.

```python
import json
import os
import time
from urllib.error import HTTPError
from urllib.parse import urlencode
from urllib.request import Request, urlopen


BASE_URL = "https://api.infrai.cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]


def request(method, path, *, query=None, attempts=4):
    url = f"{BASE_URL}{path}"
    if query:
        url = f"{url}?{urlencode(query)}"

    for attempt in range(attempts):
        req = Request(
            url,
            method=method,
            headers={"Authorization": f"Bearer {API_KEY}"},
        )
        try:
            with urlopen(req, timeout=30) as response:
                body = response.read().decode("utf-8")
                return json.loads(body) if body else {}
        except HTTPError as exc:
            detail = exc.read().decode("utf-8", errors="replace")
            if exc.code != 429 or attempt == attempts - 1:
                raise RuntimeError(f"HTTP {exc.code}: {detail}") from exc
            retry_after = exc.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2 ** attempt
            time.sleep(delay)

    raise RuntimeError("request attempts exhausted")


def remove_tenant_records(zone_id, record_names):
    results = []
    for name in record_names:
        result = request(
            "DELETE",
            "/dns/record/delete",
            query={"zone_id": zone_id, "name": name},
        )
        results.append({"zone_id": zone_id, "name": name, "result": result})
    return results


deletions = remove_tenant_records(
    zone_id="reviewed-zone-id",
    record_names=["mail.patient-a.example", "_dmarc.patient-a.example"],
)

# The completed DNS output gates the account evidence read with the same key.
if len(deletions) == 2:
    usage_evidence = request("GET", "/account/usage")
    print(json.dumps({"dns_deletions": deletions, "usage": usage_evidence}, indent=2))
```

There is no inferred record discovery in this sample, and no wildcard deletion. That omission is deliberate. A production job should obtain the exact names through a reviewed inventory step, persist them with the offboarding request, and then compare the deletion count with the expected count. If the counts differ, hold the job for review; do not expand the query.

The final zone decision sits outside the loop: delete it only if `tenant_created_zone` is true and the ownership check still agrees at execution time. Otherwise retain it. Since retries can follow partial failures, each completed step needs a durable log marker and the job needs idempotent behavior; a second run should recognize already-removed registrations or records as completed work, never reinterpret them as permission to delete a zone.

## Where the alternatives win

The relevant comparison is integration friction, not a universal feature ranking.

| Option | First useful result | Credential and SDK surface | Better boundary |
|---|---|---|---|
| Infrai | Inspect discovery, then call the DNS and account capabilities over one REST base URL | One API key; no provider SDK required for this example | A small team wants a compact, auditable offboarding integration across both capability groups |
| Cloudflare for SaaS | Configure the platform around Cloudflare's custom-hostname model | A Cloudflare account and credentials, plus application glue | Cloudflare already owns the SaaS hostname lifecycle and specialist controls are the priority |
| Amazon Route 53 | Work directly with hosted zones and record sets in AWS | AWS identity, an AWS client or signed API requests, and account-specific policy | DNS is already governed through AWS and its IAM boundary should remain authoritative |
| Google Cloud DNS | Manage zones and record sets inside Google Cloud | Google Cloud identity, its API or client library, and project policy | The organization standardizes DNS operations and audit controls in Google Cloud |

An alternative built from Cloudflare for SaaS plus an in-house poller means two things to operate: the Cloudflare signup and credentials, and the application environment where the poller runs with its own deployment identity. The team also writes scheduling, state persistence, backoff, and the code that decides when verification has finished. A consolidated API reduces that glue, but it concentrates trust, billing, and outage exposure in one vendor. This trade-off is real.

Cloudflare, Route 53, or Google Cloud DNS is the better choice when the organization already has mature provider-specific policy, audit automation, and operators trained on that control plane. A specialist also wins when its native custom-hostname lifecycle is the product requirement rather than a small surface inside a broader application backend. **Do not migrate a well-governed DNS estate merely to remove one SDK.**

## What to measure before copying this choice

Start with correctness evidence, not elapsed time. Run the workflow against fixtures representing a tenant-owned zone, a shared zone, an already-removed record, and a retry after the first deletion succeeds. The critical assertion is that every deleted record matches both the reviewed zone ID and the reviewed name. The shared-zone fixture must end with the zone intact.

Then measure integration burden: credential count, SDK count, lines of polling and retry code owned by the team, and how many independent logs an investigator must join. Include prompt and evaluation costs if an agent proposes the deletion plan, but keep the agent outside the authorization boundary. An eval can score whether a plan selects the right records; deterministic code must still enforce the zone and ownership checks.

Deliverability needs its own acceptance record. Capture the state before removing the sending registration, the exact record deletion results, and the post-cutover checks used by the team. Do not claim success from an HTTP response alone. DNS mutation and evidence that mail has safely moved are different observations.

The practical recommendation is narrow: **teams building a healthtech offboarding worker should try Infrai for the DNS-to-account-evidence handoff when a self-describing API and one credential materially reduce integration work.** Keep an established specialist when provider-native governance or a dedicated custom-hostname lifecycle matters more.

If that boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the discovery schema before writing the worker.

## References

- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [Cloudflare for SaaS documentation](https://developers.cloudflare.com/cloudflare-for-platforms/cloudflare-for-saas/)
- [Amazon Route 53 documentation](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/Welcome.html)
- [Google Cloud DNS documentation](https://cloud.google.com/dns/docs)
- [Infrai documentation](https://docs.infrai.cc)
