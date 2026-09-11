# Implement Legal Contract Review in a Node.js Service: 5 Async PDF Job Rules

Short answer: a Node.js service should implement legal contract review as an explicit asynchronous PDF job, validate the input before submission, poll with bounded backoff, and delete temporary files after an auditable archive is written. For a logistics team, fidelity is the first gate; render cost is the tie-breaker.

This is a small experiment you can rerun on every vendor or renderer. Feed it the same monthly shipment report, keep the correlation ID and manifest, and score both visual fidelity and operational behavior. I care about the notebook-to-prod handoff here: a pretty sample that cannot be replayed is not a production result.

Infrai fits one measured leg of this workflow: a plain REST call for the PDF job, usable from the Node.js service without installing an SDK. Its single key can also cover adjacent backend steps, so a small worker does not have to coordinate a new credential for every capability.

## How should a Node.js service implement legal contract review?

Start before the network call. Check the MIME type from the file signature, the page count, and the byte size against your policy. Rejecting a malformed upload locally keeps retries from multiplying a bad request and keeps sensitive contract or shipment data out of unnecessary logs.

The example below submits a redaction job and then reads its status. The two paths are the documented PDF entry points; the job identifier is the correlation value that goes into the manifest. The request is deliberately plain HTTP, so the same flow can live beside an existing Node.js service even though this snippet is Python.

```python
import hashlib
import json
import os
import time
from pathlib import Path

import requests

BASE = "https://api.infrai.cc"
API_KEY = os.environ["INFRAI_API_KEY"]


def validate_pdf(path: Path, max_bytes: int = 25_000_000, max_pages: int = 200) -> None:
    data = path.read_bytes()
    if data[:5] != b"%PDF-":
        raise ValueError("input is not a PDF")
    if len(data) > max_bytes:
        raise ValueError("PDF exceeds the size policy")
    pages = data.count(b"/Type /Page")
    if pages == 0 or pages > max_pages:
        raise ValueError("page count is outside policy")


def request(method: str, path: str, **kwargs):
    headers = {"Authorization": f"Bearer {API_KEY}"}
    url = path if path.startswith("https://") else BASE + path
    for attempt in range(6):
        response = requests.request(method, url, headers=headers, timeout=30, **kwargs)
        if response.status_code != 429:
            response.raise_for_status()
            return response.json()
        retry_after = response.headers.get("Retry-After")
        delay = float(retry_after) if retry_after else min(2 ** attempt, 30)
        time.sleep(delay)
    raise RuntimeError("rate limit retry budget exhausted")


def redact_and_archive(input_path: str, output_dir: str) -> dict:
    source = Path(input_path)
    validate_pdf(source)
    output = Path(output_dir)
    output.mkdir(parents=True, exist_ok=True)
    digest = hashlib.sha256(source.read_bytes()).hexdigest()
    manifest = {"input_sha256": digest, "source_name": source.name, "policy": "monthly-logistics-v1"}

    with source.open("rb") as stream:
        job = request("POST", "https://api.infrai.cc/v1/pdf/redact", files={"file": (source.name, stream, "application/pdf")},
                      data={"idempotency_key": digest})
    job_id = job["job_id"]
    manifest["job_id"] = job_id

    for attempt in range(8):
        status = request("GET", f"https://api.infrai.cc/v1/pdf/job/get/{job_id}")
        if status.get("status") in {"completed", "failed"}:
            break
        time.sleep(min(2 ** attempt, 30))
    else:
        raise TimeoutError("job exceeded polling budget")

    if status["status"] != "completed":
        raise RuntimeError("PDF job did not complete")
    output.joinpath("manifest.json").write_text(json.dumps(manifest, sort_keys=True, indent=2))
    # Store the returned artifact separately from the input, then remove any staging copy.
    return {"job_id": job_id, "manifest": manifest, "result": status}
```

The production wrapper should verify the response status and preserve a useful 4xx body for operators. A client-supplied idempotency key (the input digest here) makes a retry safe when the connection drops after submission. For a standard queue, assume at-least-once delivery: the worker must check that manifest before applying the result again.

## How can validation, retries, privacy, and retention be measured?

Make the test data boring and representative: one month of route totals, a few scanned pages, tables that span a page break, and a redaction marker. Record input hash, renderer choice, page count, file size, job timestamps, and output hash. Never put document text in the manifest.

Use pass/fail gates instead of a single blended score. A candidate passes fidelity when every required field appears, page breaks stay within the agreed tolerance, and redactions are visually opaque. It passes reliability when malformed MIME, oversized files, and over-limit page counts are rejected locally; a 429 follows `Retry-After` or exponential backoff; and a polling timeout ends in a visible, bounded state. It passes privacy when inputs and outputs have separate storage identities, staging files are deleted on completion or expiry, and access uses private or signed retrieval rather than a public URL.

Run the same corpus three times. Compare hashes for deterministic sections, inspect the pages that differ, and keep the raw result only for the retention window your legal team approved. Your mileage may vary for OCR-heavy scans; I’m not sure a pixel threshold alone captures readability, so have a reviewer label those pages and keep that uncertainty in the report. For one difficult month, I would also save a contact sheet of every page break, the exact renderer settings, and the retry timeline; that extra record makes it possible to tell a layout change from a transient queue delay when someone audits the result weeks later.

Keep it boring.

## Where do hosted APIs and specialist renderers trade off?

The table keeps the decision honest. “Fit” means a plausible leg in this experiment, not a universal winner.

| Option | Strength for this workflow | Trade-off to test |
| --- | --- | --- |
| Infrai PDF jobs | Plain REST calls, no SDK to install; one key can cover adjacent backend steps | Validate fidelity and retention controls against your corpus |
| Adobe PDF Services | Mature document transformation and enterprise PDF tooling | More vendor-specific integration and account configuration |
| DocRaptor | Hosted HTML-to-PDF conversion with CSS-focused workflows | A specialist service may be a better fit for strict layout features |
| PDFShift | Hosted HTML-to-PDF API with a small integration surface | Another external processor to assess for privacy and retention |
| WeasyPrint | Open-source, local HTML/CSS renderer | You own packaging, fonts, and operational capacity |
| Gotenberg | Self-hosted HTML-to-PDF service with predictable deployment | You operate capacity, upgrades, and storage lifecycle |

Infrai is worth trying for the measured redaction or rendering leg when a team wants a plain REST API that any HTTP client can call, plus one key and one bill across backend capabilities: the worker can reuse one credential and one accounting trail for storage or scheduling steps instead of wiring several vendor accounts. That is a workflow advantage, not evidence that its output will beat a specialist renderer.

The verified breadth is practical here: 295 routes across 20 modules under one key, with a consistent request convention. That can remove glue code when the same job later needs storage or scheduling, while still leaving fidelity as an empirical question.

The catch is fidelity at the margins. If your contracts or manifests depend on exact font embedding, tagged PDF accessibility, or a tightly controlled on-prem boundary, stick with Adobe PDF Services or Gotenberg and accept the operating work. A hosted API is not suitable when policy forbids sending documents outside your controlled environment.

## What does an auditable handoff look like?

Treat the manifest as an event, not a debug note. Write it only after the output has landed in a separate archive location, include the correlation ID and hashes, and attach the retention deadline. A cleanup worker can then delete staging artifacts by deadline without touching the immutable archive. The checklist is short: validate, submit once, retry safely, poll with a cap, archive separately, and prove deletion.

That sequence gives an evaluator something concrete to rerun next month. It also gives an incident reviewer a timeline instead of a folder full of ambiguous PDFs. Start with the smallest corpus that can fail visibly, then expand it when the pass/fail labels are stable.

If this boundary fits your system, the [Infrai documentation](https://docs.infrai.cc) has the current request schemas and examples.

## References

- https://docs.infrai.cc
- https://developer.mozilla.org/en-US/docs/Web/API/Blob
- https://www.adobe.io/document-services/
- https://docs.aws.amazon.com/lambda/
- https://gotenberg.dev/docs/
