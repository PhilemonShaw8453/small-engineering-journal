# Resume Extraction at Batch Scale: Stable JSON Records for Applicant Tracking

TL;DR: The best API approach is a bounded batch pipeline, not a synchronous endpoint that promises one perfect resume object. Preserve the source PDF, extract evidence into a versioned intermediate record, validate the final JSON, and route uncertain fields for review. Select an implementation by sustained throughput at an acceptable review rate, measured on the resumes the applicant-tracking system actually receives.

That decision rule also fits a customer-support team rendering its monthly report to PDF and archiving it: bursts are normal, order matters, and an accepted job is not the same as a completed artifact. Resume ingestion adds a harder semantic step, but the operational shape is familiar. A queue absorbs the burst; bounded workers protect downstream capacity; immutable inputs and explicit job states make reruns explainable.

The tempting first design is a request that accepts a PDF, extracts text, infers fields, and returns JSON before the connection closes. It is wonderfully easy to demonstrate in a notebook. Under a monthly batch, though, slow documents occupy request capacity, retries duplicate work, and one ambiguous date can be mistaken for a successful parse. The production choice should separate submission from processing and make ambiguity visible.

## How should an API parse a PDF resume into structured JSON?

PDF standardizes a document representation, not a resume's business meaning. A page can look obvious to a recruiter while its underlying content order does not match the visual reading order. Other inputs may contain only page imagery, unusual fonts, tables, sidebars, or repeated headers. A parser therefore needs to retain what it observed rather than jumping directly from document bytes to an authoritative applicant profile. Use three representations. First, keep the original bytes with a content digest and an access-controlled archive reference. Second, create an evidence layer containing page number, extracted span, location, and extraction method. Third, map that evidence into the applicant-tracking contract. This separation lets a team replace extraction or field inference without silently changing the meaning of previously accepted records. The contract should distinguish absence from uncertainty. `null` can mean that no value was found; a confidence value and evidence reference can explain why a present value still needs review. Do not turn confidence into fake precision. Its useful role is routing: accept, review, or reject according to thresholds calibrated on a labeled evaluation set.

Evidence comes first.

## Treat batch throughput as a flow-control problem

Throughput is not the fastest single-document latency. For a batch of 10,000 resumes, measure completed documents per minute after warm-up, while enforcing the same validation and review policy used in production. Record the p50 and p95 completion latency too, but do not let a fast median hide a queue that never drains.

Bursts expose shortcuts.

A useful experiment compares the simple synchronous path with an asynchronous path under identical inputs. Fix the worker limit, then increase it deliberately: 4, 8, 16, and 32 is a reasonable experiment grid, not a universal prescription. At every step, capture queue age, processing time by stage, failure category, peak memory, output-schema violations, and the percentage sent to review. Stop increasing concurrency when throughput flattens, review quality degrades, or a protected dependency reaches its operating limit.

Backpressure is part of correctness. Submission should return a stable job identifier after durable acceptance, while processing moves through explicit states such as `queued`, `running`, `needs_review`, `completed`, and `failed`. A retry must carry the same content digest and parser configuration version. That makes idempotency a property of the job, rather than a hope attached to an HTTP timeout.

This is the key trade-off: bounded workers may leave some compute idle during a quiet hour, but they keep memory and dependent services predictable during a monthly burst. Predictability wins here.

## A focused Python boundary

The following example starts after a PDF adapter has produced evidence spans. That adapter is intentionally outside the core: native text extraction, optical character recognition, and mixed-document handling can change independently. The code concentrates on the durable boundary that an applicant-tracking integration owns.

```python
from __future__ import annotations

import asyncio
import hashlib
from dataclasses import asdict, dataclass
from typing import Awaitable, Callable, Literal


@dataclass(frozen=True)
class Evidence:
    page: int
    text: str
    method: Literal["embedded_text", "ocr"]


@dataclass(frozen=True)
class CandidateRecord:
    schema_version: str
    source_sha256: str
    full_name: str | None
    email: str | None
    skills: list[str]
    evidence: list[Evidence]
    review_reasons: list[str]


Extract = Callable[[bytes], Awaitable[list[Evidence]]]
Infer = Callable[[list[Evidence]], Awaitable[dict[str, object]]]


async def parse_resume(
    pdf: bytes,
    extract: Extract,
    infer: Infer,
) -> CandidateRecord:
    evidence = await extract(pdf)
    fields = await infer(evidence)

    name = fields.get("full_name")
    email = fields.get("email")
    skills = fields.get("skills", [])
    review_reasons: list[str] = []

    if not isinstance(name, (str, type(None))):
        review_reasons.append("invalid_full_name_type")
        name = None
    if not isinstance(email, (str, type(None))):
        review_reasons.append("invalid_email_type")
        email = None
    if not isinstance(skills, list) or not all(isinstance(x, str) for x in skills):
        review_reasons.append("invalid_skills_type")
        skills = []
    if not evidence:
        review_reasons.append("no_extractable_evidence")

    return CandidateRecord(
        schema_version="1.0",
        source_sha256=hashlib.sha256(pdf).hexdigest(),
        full_name=name,
        email=email,
        skills=sorted(set(skills)),
        evidence=evidence,
        review_reasons=review_reasons,
    )


async def parse_batch(
    documents: list[bytes],
    extract: Extract,
    infer: Infer,
    worker_limit: int = 8,
) -> list[dict[str, object]]:
    semaphore = asyncio.Semaphore(worker_limit)

    async def bounded(pdf: bytes) -> dict[str, object]:
        async with semaphore:
            record = await parse_resume(pdf, extract, infer)
            return asdict(record)

    return await asyncio.gather(*(bounded(pdf) for pdf in documents))
```

This is deliberately small. In a deployed service, the input list would normally be job references fetched in pages, and results would be persisted individually rather than held until every task finishes. The semaphore still captures the important mechanism: a batch can be large without making downstream concurrency unbounded.

There is one trap in the example worth calling out. `asyncio.gather` preserves input order, which is convenient for an experiment, but it also waits for all submitted tasks and propagates failures unless handling is added. Production workers should isolate each job, persist its terminal state, and retry only failures classified as transient. Bad files and schema violations need a dead-letter or review path, not an endless retry loop.

## Evaluate the record, not just the extracted text

Start with a frozen corpus that reflects the real intake: native PDFs, scanned pages, multiple columns, long employment histories, and sparse early-career resumes. Remove or protect personal data according to the organization's data policy. Label the fields that drive applicant-tracking workflows, then keep a separate holdout set for release decisions.

Field-level scoring is more actionable than one document score. Exact matching can work for normalized email addresses. Names may need a documented normalization policy. Employment entries need tests for grouping, ordering, open-ended dates, and evidence linkage. Skills require a declared rule about whether the parser may normalize synonyms or must preserve source wording. Each rule changes what “correct” means, so it belongs in the evaluation contract.

Measure these together:

- required-field precision and recall on the holdout set;
- schema-valid output rate and evidence coverage;
- review rate under the chosen routing thresholds;
- sustained documents per minute and oldest-job age;
- permanent failures, transient retries, and duplicate suppression;
- compute and model-token consumption per completed document.

The failed/simple approach often looks competitive if the test reports only text extraction speed. It loses its advantage once the experiment charges it for invalid records, repeated requests, and human correction. A prompt change should face the same harness as a parser change: pin its version, run the holdout set, compare field errors and token use, and deploy only when the combined quality and throughput budget still holds.

## Make the archive replayable

Store enough metadata to reproduce a record: source digest, schema version, extraction version, inference or prompt version, completion timestamp, review disposition, and evidence links. Keep the original document under retention and access rules appropriate for applicant data. Logs should use job identifiers and failure categories; dumping resume text into general application logs creates an unnecessary second data store.

Schema evolution needs an explicit choice. Additive fields can often coexist within a version, while changed meanings require a new version and a migration plan. Never reinterpret an archived JSON object in place. Reprocessing should create a new result linked to the same source digest so auditors can see what changed and why.

Before copying this design, measure the actual arrival curve and the downstream applicant-tracking write limit. Then choose a worker bound that drains the peak batch inside the required window without increasing the review rate. The winning approach is the one that produces traceable, schema-valid candidate records at sustained load. A flashy one-file demo is not that test.

## Further reading

- ISO 32000-2, Portable Document Format: https://www.iso.org/standard/75839.html
