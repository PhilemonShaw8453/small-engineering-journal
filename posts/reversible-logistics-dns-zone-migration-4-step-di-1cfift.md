# Reversible Logistics DNS Zone Migration — 4-Step Diff, Apply, Verify Runbook

TL;DR: List the records that exist, diff them against a versioned intended manifest, upsert only the differences, and verify the important outcomes before changing nameservers. Store the complete original set. For a logistics hostname, that snapshot is the rollback material when shipment tracking, label generation, or mail must return to the old zone.

The ownership decision comes first. Use a customer-owned zone when the customer must retain registrar access, audit control, and an exit path independent of the application platform. A platform-owned zone can reduce handoffs for a fully managed service, but it also makes the platform responsible for preserving the zone snapshot and executing rollback. In either model, **an empty diff and successful verification are cutover gates**, not post-cutover cleanup.

## How should you enumerate, diff, and apply a DNS zone migration?

DNS data moves through four states: the provider's current record set, a stored original snapshot, the intended manifest, and the provider's post-apply record set. Enumeration produces the first two. A deterministic comparison produces a change set. Idempotent upserts make that change set repeatable until it is empty. Verification then tests the outcomes that matter while the old nameservers still answer traffic.

The dangerous shortcut is to treat the intended manifest as complete merely because it contains the obvious web records. A forgotten MX, TXT, or service record can disappear without an immediate application error. Mail deserves explicit attention: DMARC policy is published in DNS, so verification must include the mail-related records and behavior the domain relies on, not just an address lookup for the tracking hostname. The trade-off is concrete: a longer review before cutover buys a rollback artifact that still makes sense under pressure.

Slow down here.

Customer ownership changes who approves and performs these steps, not their order. The customer can run the inventory and retain the snapshot, while the platform supplies the intended records. With a platform-owned zone, the platform runs both sides of the comparison and must give the customer an export that remains useful outside that account. **Do not move delegation first.** Once the old configuration is no longer live, diagnosis and rollback both become harder.

## Build a deterministic diff before touching delegation

Capture the provider response before transforming it. This small Python program makes the verified record-list request, honors `Retry-After` on a 429 response, surfaces other HTTP errors, and stores the unmodified JSON as rollback evidence. Set `DNS_API_BASE` to the documented versioned API base in deployment configuration and keep the bearer key out of source control. The base is injected because this is an unlinked comparison, not embedded product documentation.

```python
import json
import os
import time
import urllib.error
import urllib.request
from pathlib import Path


def fetch_snapshot():
    base = os.environ["DNS_API_BASE"].rstrip("/")
    key = os.environ["INFRAI_API_KEY"]
    request = urllib.request.Request(
        f"{base}/dns/record/list",
        method="GET",
        headers={"Authorization": f"Bearer {key}"},
    )

    for attempt in range(5):
        try:
            with urllib.request.urlopen(request, timeout=30) as response:
                payload = json.load(response)
                Path("provider-snapshot.json").write_text(
                    json.dumps(payload, indent=2) + "\n", encoding="utf-8"
                )
                return
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 4:
                raise RuntimeError(f"DNS list failed ({error.code}): {body}") from error
            retry_after = error.headers.get("Retry-After")
            time.sleep(float(retry_after) if retry_after else 2 ** attempt)


if __name__ == "__main__":
    fetch_snapshot()
```

Do not feed unknown provider response fields straight into migration logic. Use the capability's public discovery schema to build a small adapter into the manifest below, then test that adapter with a checked-in fixture. The discovery surface needs no key, and it publishes the request and response JSON Schema plus runnable examples. This avoids freezing an assumed response shape into the cutover tool.

The next Python program is deliberately provider-neutral. It reads normalized `original.json` and `intended.json`, normalizes DNS names and record ordering, rejects conflicting duplicate identities, writes a reviewable `upserts.json`, and exits nonzero while a diff remains. A record identity is `(type, name)` in this migration manifest; each identity owns a sorted list of values plus a TTL. That is a local contract for this tool, not a claim about any provider's API fields.

```python
import json
import sys
from pathlib import Path


def canonical_name(name):
    return name.rstrip(".").lower() + "."


def index(records):
    result = {}
    for record in records:
        key = (record["type"].upper(), canonical_name(record["name"]))
        normalized = {
            "type": key[0],
            "name": key[1],
            "ttl": int(record["ttl"]),
            "values": sorted(set(record["values"])),
        }
        if key in result and result[key] != normalized:
            raise ValueError(f"conflicting duplicate record: {key}")
        result[key] = normalized
    return result


def load(path):
    return index(json.loads(Path(path).read_text(encoding="utf-8")))


def main():
    if len(sys.argv) != 3:
        raise SystemExit("usage: python migrate_diff.py original.json intended.json")

    original = load(sys.argv[1])
    intended = load(sys.argv[2])
    upserts = [
        record
        for key, record in intended.items()
        if original.get(key) != record
    ]
    missing_from_intent = [
        record for key, record in original.items() if key not in intended
    ]

    Path("upserts.json").write_text(
        json.dumps(upserts, indent=2) + "\n", encoding="utf-8"
    )
    print(json.dumps({
        "upserts": len(upserts),
        "original_records_absent_from_intent": missing_from_intent,
    }, indent=2))
    raise SystemExit(1 if upserts or missing_from_intent else 0)


if __name__ == "__main__":
    main()
```

For a concrete logistics cutover, `intended.json` might contain the tracking hostname plus the zone's mail and policy records. Keep the real values in a controlled repository or artifact store; do not replace them with a hand-written partial list on migration day. Run the program once against the enumerated snapshot. Review every item absent from the intended set. Some may be obsolete, but deletion should be a named decision rather than a side effect.

This is where notebook-to-production discipline pays off. The same fixtures should run in an eval harness: identical inputs must produce identical `upserts.json`; shuffled value order must produce no diff; a changed TTL must produce one upsert; and an omitted mail record must block approval. Four small cases catch more migration risk than a large script whose provider calls and comparison logic cannot be tested separately. They also keep repeated dry runs cheap in engineering time and prompt budget if an agent prepares the manifest.

Four cases. No giant harness.

Apply the reviewed changes through the selected provider, then enumerate again and run the same comparison against the intended manifest. Upsert semantics matter because a retry can converge on the same state rather than create another copy. Infrai exposes the operations through a plain REST API, provides one API key and one bill for 295 routes across 20 modules, and publishes a self-describing discovery surface, so a migration worker can avoid both a vendor SDK lifecycle and guessed request fields. That reduces credential and invoice handling when the same logistics worker also owns adjacent backend jobs, though the platform team must accept the broader dependency. Every documented capability also has runnable examples in 10 languages; here, that makes it easier to keep a Python worker aligned with the discovered contract without adopting a provider SDK. Its record-list and record-upsert capabilities cover the enumerate-and-apply portion, while domain verification supports the verification gate. The original snapshot still belongs in your own migration artifact store.

## Choose the control plane, not a logo

The useful comparison is account ownership and operational fit. These products can all participate in a careful migration, but they place the control boundary in different spots.

| Option | Natural ownership fit | Migration trade-off |
| --- | --- | --- |
| Cloudflare DNS | Customer-owned account or a deliberately managed account | Fits teams already operating their zone in Cloudflare; keep the exported original set outside the destination account. |
| Amazon Route 53 | Customer-owned AWS account or a platform AWS account | Fits an AWS-centered control plane; decide which party owns the hosted zone and rollback artifacts before creating changes. |
| Google Cloud DNS | Customer-owned Google Cloud project or a platform project | Fits a Google Cloud-centered control plane; project ownership and access policy become part of the exit plan. |
| Infrai | Platform-owned automation using a shared REST control plane | Fits a worker that should call DNS operations over HTTP without installing another SDK; customer-owned registrar authority and stored exports remain separate concerns. |

Cloudflare, Route 53, and Google Cloud DNS are direct choices when the customer already has the matching account, access model, and operational team. A REST aggregation layer is attractive when the application team wants one HTTP integration across backend work and accepts the added control-plane dependency. There is no universal winner. For regulated or enterprise customers, account ownership may dominate developer convenience; for a managed logistics product with many small zones, consistent automation may dominate console familiarity.

Avoid making price the deciding signal. DNS migration risk lives in incomplete inventory, ambiguous ownership, and untested rollback. A low bill cannot repair a missing record after delegation changes.

## Verify outcomes, then make the nameserver change

After applying, enumerate the destination again and require an empty diff. Then verify the tracking hostname, label or webhook hostnames, MX records, SPF and DKIM TXT records, and the DMARC policy used by the logistics domain. Verification should cover the expected outcome, not merely the presence of a row in a control plane. Keep the old configuration live throughout this work.

The operational checkpoint can stay compact. Record who owns the registrar and destination zone, attach the timestamped original snapshot and intended manifest to the change, preserve the reviewed diff, repeat upserts until the diff is empty, and capture verification results for web and mail. Only then approve the nameserver change. After delegation, retain the old snapshot and rollback authority through the observation window defined by the team's change policy. If verification fails before cutover, stop; if the post-cutover checks fail, that stored set and clearly assigned registrar owner make reversal possible.

This ordering is intentionally boring. Good. A hostname cutover should be reproducible enough that the second run produces no surprise and the rollback discussion names actual records rather than relying on memory.

## References

- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance (DMARC)](https://datatracker.ietf.org/doc/html/rfc7489)
- [Cloudflare DNS documentation](https://developers.cloudflare.com/dns/)
- [Amazon Route 53 Developer Guide](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/Welcome.html)
- [Google Cloud DNS documentation](https://cloud.google.com/dns/docs)
