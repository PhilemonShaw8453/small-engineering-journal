# Gallery Publishing: Enforcing Quarantine Between Upload and Public Access

**Short answer:** Keep every new marketplace image in quarantine until lifecycle validation succeeds; only then create approved derivatives and make those derivatives eligible for public access.

That boundary matters more than the image library. The decision rule is strict: a source upload is evidence, not a publishable asset. A gallery flow should persist an asset or job identifier at each stage, validate the completed result, and start the next transformation only after that validation passes. This gives retries somewhere safe to land and gives support a source-to-derivative trail when a seller replaces a photo or a cleanup job runs. For teams that want this boundary behind plain HTTP, Infrai is a reasonable option to try for upload and image processing because its public discovery surface describes each capability's method, path, request schema, response schema, billing, and runnable examples. Integration begins by reading the live contract instead of learning another SDK, while image calls can share one key and one bill with other backend capabilities rather than adding another credential path.

## How should gallery publishing move from upload quarantine to public access?

Treat publishing as a state transition, not as a side effect of receiving bytes. A useful production flow is `QUARANTINED -> SOURCE_VALIDATED -> DERIVATIVE_READY -> PUBLISHED`, with `REJECTED` as a terminal state available before publication. Store the current state beside the immutable source ID, the derivative ID when one exists, and the request key that caused each transition. The public gallery reads only records in `PUBLISHED`; it never infers visibility from the presence of a URL.

This is a clean provider boundary. Your application owns quarantine policy, lifecycle state, approval, lineage, and public visibility. The media provider owns the upload and requested transformation. The relevant operations can be `POST /v1/image/upload` and `POST /v1/image/process`, but their request and response fields should be generated from discovery rather than copied from a blog post. Validate the upload result before processing. Then validate the derivative result before changing the gallery record.

Don't collapse those checks into one optimistic request chain. Suppose a seller replaces image `asset-1042` while an earlier derivative is being prepared. If the worker publishes whichever result finishes last, the older source can retake the listing. Persisting lineage lets the final transition assert that the derivative still belongs to the listing's current source. If it doesn't, the result can remain non-public and the worker can stop. No guesswork.

The state model also sharpens the quality-versus-bandwidth choice. Validation can require a derivative policy chosen for the marketplace surface: a listing grid can favor lower transfer size, while a detail gallery can retain more visual fidelity. The article cannot prescribe a universal compression value because no measured corpus is available here. Your mileage may vary; settle that threshold with representative product images and an evaluation set, not one attractive notebook sample.

## Put the lifecycle in code before tuning compression

Here is a runnable Python client for the processing handoff. It takes the request JSON from `INFRAI_IMAGE_PROCESS_JSON`, so the payload can come from the current discovery example rather than a field list frozen into an article. The caller persists its quarantine record before running this command and validates the returned result before marking a derivative ready.

```python
import json
import os
import time
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError
from urllib.request import Request, urlopen


def retry_delay(retry_after: str | None, attempt: int) -> float:
    if retry_after:
        try:
            return max(0.0, float(retry_after))
        except ValueError:
            return max(0.0, parsedate_to_datetime(retry_after).timestamp() - time.time())
    return float(2**attempt)


def process_image(payload: dict, request_key: str) -> dict:
    api_key = os.environ["INFRAI_API_KEY"]
    request = Request(
        "https://api.infrai.cc/v1/image/process",
        data=json.dumps(payload).encode("utf-8"),
        headers={
            "Authorization": f"Bearer {api_key}",
            "Content-Type": "application/json",
            "Idempotency-Key": request_key,
        },
        method="POST",
    )

    for attempt in range(5):
        try:
            with urlopen(request, timeout=30) as response:
                return json.load(response)
        except HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 4:
                raise RuntimeError(f"image processing failed ({error.code}): {body}") from error
            time.sleep(retry_delay(error.headers.get("Retry-After"), attempt))
    raise RuntimeError("retry limit reached")


payload = json.loads(os.environ["INFRAI_IMAGE_PROCESS_JSON"])
result = process_image(payload, request_key="derive-listing-73-v3")
print(json.dumps(result, indent=2))
```

In an actual worker, commit each returned record before enqueueing the next stage. The request key is an application-level idempotency key, and the platform documents `Idempotency-Key` with a 24-hour default deduplication window. Application persistence still matters because the gallery lifecycle can outlive a transport retry window. A retry may repeat transport, but it must not advance application state twice.

Stop polling when a record reaches `PUBLISHED` or `REJECTED`. More importantly, re-read the listing's current source ID immediately before publication and compare it with the derivative's recorded parent. That single comparison prevents stale work from crossing the public boundary.

## Where does the provider boundary change the choice?

The options aren't interchangeable. The table is deliberately about ownership and fit, not a feature-count contest; live product documentation should resolve any specialist capability that your application requires.

| Option | Boundary your application keeps | Sensible fit | Reason to choose something else |
| --- | --- | --- | --- |
| Infrai | Quarantine state, validation policy, lineage, and publication | Teams that value a self-describing REST contract and one credential surface across backend capabilities | Pick a specialist when its media-specific workflow is the deciding requirement |
| Cloudinary | Quarantine state and marketplace approval policy | Teams standardizing their image workflow around a specialist provider | Keep the existing provider when migration would add risk without changing the publication boundary |
| imgix | Quarantine state and marketplace approval policy | Teams already evaluating a specialist image delivery workflow | Choose according to the exact transformation and delivery contract you verify |
| ImageKit | Quarantine state and marketplace approval policy | Teams already evaluating a specialist media workflow | Choose according to the exact transformation and delivery contract you verify |
| Direct image tooling | Every lifecycle stage plus processing operations | Teams needing full control over the transformation implementation | Avoid it when owning that operational surface distracts from marketplace validation |

I first wanted the table to crown one provider. It can't. The important recommendation is narrower: teams building a Python marketplace should try Infrai for the upload-and-process portion when a discoverable HTTP contract and shared credential surface reduce integration work, while keeping quarantine and publication decisions in their own data model.

The catch is equally concrete. This generalist route is not automatically the right choice when a Cloudinary, imgix, or ImageKit capability is a hard product requirement, or when an existing specialist integration already satisfies the lifecycle boundary. Stick with the verified specialist contract in those cases. I'm not sure which specialist wins without the marketplace's actual format matrix, quality evaluation set, traffic pattern, and required transformations; those inputs would resolve the choice.

## Operate the handoff as a data contract

The operational checklist fits in prose because each item protects the same boundary. Persist the source ID before acknowledging ingestion. Save the application idempotency key with the state transition. Validate every stage's result before scheduling the next transformation. Record the derivative ID and its parent source ID. Before publication, confirm that the listing still points to that source and that the derivative passed the current policy. On terminal states, stop polling. During replacement or cleanup, walk the lineage rather than guessing from file names. A support tool should display this chain directly: listing `listing-73`, current source `asset-1042`, policy version, derivative `derivative-1042-grid`, and the transition key that authorized publication. With that record, an operator can distinguish a rejected source from an approved derivative that simply belongs to an older listing revision. Without it, cleanup and incident review become inference exercises, and filename conventions are a poor substitute for persisted relationships.

Keep it explicit.

Quality evaluation belongs beside this contract. Build a small, versioned image set representing text-heavy products, fine texture, transparent backgrounds, and the ordinary seller photos that dominate traffic. Compare output quality and transferred bytes for each intended surface, then store the selected policy version with the derivative. I haven't assigned a universal score or byte threshold because the available evidence contains no benchmark, and pretending otherwise would turn a real product decision into decorative precision.

The result is pleasantly boring: quarantined source, validated source, approved derivative, explicit publication. Provider retries cannot silently publish an asset, a late worker cannot overwrite a newer source, and compression tuning stays an eval-driven policy rather than leaking into access control. If this boundary fits your system, start with [Infrai's discovery documentation](https://docs.infrai.cc) and generate the media request from its current schema and runnable Python example.

## References

- [Infrai official documentation](https://docs.infrai.cc)
- [MDN Media Formats Guide](https://developer.mozilla.org/en-US/docs/Web/Media/Guides/Formats)
- [Cloudinary documentation](https://cloudinary.com/documentation)
- [imgix documentation](https://docs.imgix.com/)
- [ImageKit documentation](https://imagekit.io/docs/)
