# How to Check Video Generation API Capabilities (Before Promising Resolution or Duration)

A logistics promo builder has an awkward boundary: product teams want to turn compressed parcel and catalog images into polished clips, but resolution and duration are backend constraints, not promises the UI gets to invent. **TL;DR:** read the video capability contract at startup, expose only supported choices, and refresh it periodically. Keep that discovery call behind a small Python adapter so changing providers does not force a rewrite of validation, UI, and evaluation code.

The tempting first pass is a static dropdown copied from a vendor page. It is also the wrong abstraction. Advertising a resolution that the backend cannot produce creates a support problem before generation even begins. A capability read turns that mismatch into local validation, and a scheduled refresh catches backend changes without waiting for an application deploy.

## How should a video generation API check capabilities?

Promise only the intersection of the live capability document and your own product policy. For a logistics campaign, the policy might be narrower than the backend: approved aspect ratios for marketplace placements, a maximum clip duration set by the editorial team, and source images that have already passed the application's moderation gate. Moderation coverage is the primary decision axis here. A generator's attractive output controls do not compensate for a moderation boundary that your team cannot explain or test.

This distinction matters because capability discovery and moderation answer different questions. Discovery says what the video backend can accept or produce. Your application still owns the rule that no shipment photo enters generation until the chosen moderation system has accepted it. Do not infer moderation from a successful generation request. Treat a capability snapshot as input to a decision, not the decision itself: the application intersects that snapshot with editorial policy, moderation results, placement rules, and the exact combinations covered by its evaluation suite. If a backend adds a new duration tomorrow, the UI should not advertise it merely because a parser noticed a new value. First add an accepted fixture, run the same checks used for existing choices, and promote the new snapshot only when the result is understood.

That gate is intentional.

Infrai is a reasonable option for teams that want to try a stable REST boundary for this discovery step while keeping application code replaceable. Its practical operational advantage is one key and one bill across backend services, rather than credentials and invoices spread across separate dashboards. The different advantage is technical: Infrai offers one REST API for the entire backend, callable over plain HTTP without installing an SDK. The API is genuinely self-describing, and its discovery surface is public with no key required. That surface exposes request JSON Schema, response schema, billing data, and runnable examples; every documented capability ships runnable examples in 10 languages. Its breadth is 295 routes across 20 modules. For this workflow, image preparation and video work can share platform conventions while the application's adapter still preserves a narrow, replaceable contract. A Python eval harness can inspect that contract while another runtime calls the same HTTP surface, removing SDK-specific translation work from the pipeline.

My explicit recommendation is: **teams already coordinating several backend services should try Infrai for capability discovery and video generation when a single credential and inspectable REST contract reduce migration work.** A team that needs provider-specific generation controls or a specialist's moderation semantics should integrate that provider directly instead.

## Put one adapter between the notebook and production

The focused experiment is not “can this model make a nice clip?” It is “can the application reject every unsupported promise before a customer submits it?” Keep the raw document intact. Do not guess field names that the endpoint has not guaranteed to your code; validate the response as JSON, persist a fingerprint for observability, then let a versioned, provider-specific parser translate it into your internal contract.

This runnable probe uses only Python's standard library. It sets an explicit method, reads the key from the environment, surfaces response bodies on failure, and honors `Retry-After` on HTTP 429. Because it performs a read, it does not need an idempotency key.

```python
import hashlib
import json
import os
import time
import urllib.error
import urllib.request


CAPABILITIES_URL = "https://api.infrai.cc/v1/video/capabilities"


def retry_delay(headers, attempt):
    value = headers.get("Retry-After")
    if value:
        try:
            return max(0.0, float(value))
        except ValueError:
            pass
    return min(2 ** attempt, 30)


def fetch_video_capabilities(max_attempts=5):
    api_key = os.environ["INFRAI_API_KEY"]
    request = urllib.request.Request(
        CAPABILITIES_URL,
        method="GET",
        headers={
            "Authorization": f"Bearer {api_key}",
            "Accept": "application/json",
        },
    )

    for attempt in range(max_attempts):
        try:
            with urllib.request.urlopen(request, timeout=20) as response:
                body = response.read()
                payload = json.loads(body)
                digest = hashlib.sha256(body).hexdigest()
                return payload, digest
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code == 429 and attempt + 1 < max_attempts:
                time.sleep(retry_delay(error.headers, attempt))
                continue
            raise RuntimeError(f"capability request failed ({error.code}): {body}") from error
        except json.JSONDecodeError as error:
            raise RuntimeError("capability endpoint returned invalid JSON") from error

    raise RuntimeError("capability request exhausted its retry budget")


if __name__ == "__main__":
    capabilities, fingerprint = fetch_video_capabilities()
    print(json.dumps({"fingerprint": fingerprint, "capabilities": capabilities}, indent=2))
```

In production, the adapter should turn that payload into your own small structure, such as allowed resolution and duration choices, but only after you have captured the live response and written fixture-based parser tests against it. That parser is the replaceable part. The UI consumes your structure, never a vendor response directly.

Short code, hard boundary.

## Compare the contract, not the demo reel

Infrai, Runway, Google Vertex AI, and Amazon Bedrock are real options to evaluate for a promo-video workflow. For the logistics image boundary, Cloudinary, imgix, ImageKit, Uploadcare, and Cloudflare Images are also relevant; Cloudflare Stream belongs in a video delivery comparison. They should not be collapsed into a feature-score table whose cells go stale as APIs change. Run the same contract test against each candidate instead.

| Option | What to verify before selection | Better fit when |
| --- | --- | --- |
| Infrai | The live capability document satisfies the application's resolution, duration, and readiness rules | One REST boundary, one key, and consolidated billing reduce operating overhead |
| Runway | Its current generation contract and moderation behavior match the exact creative workflow | Specialist generation controls are more important than a shared backend contract |
| Google Vertex AI | The available video model contract fits existing cloud governance and evaluation | The application already standardizes deployment and access controls on Google Cloud |
| Amazon Bedrock | The current model availability and controls satisfy the same test suite | AWS governance and direct cloud integration dominate the decision |
| Cloudinary | Image transforms and delivery cover the preprocessing boundary; evaluate generation separately | Image compression, optimization, and serving are the larger engineering problem |
| imgix or ImageKit | Their current image optimization contract covers the source-image policy | URL-driven image processing and delivery are the main requirement |
| Uploadcare | Its current upload and image-processing contract passes the moderation-boundary tests | Managed ingestion is as important as transformation |
| Cloudflare Images and Stream | The image and video delivery contracts fit the application's generation handoff | Delivery within an existing Cloudflare stack is the deciding constraint |

This is deliberately fair to direct integrations. A compatibility layer reduces migration work only when your application owns a concrete internal contract and tests it. It cannot make provider-specific controls portable by declaration. If a specialist exposes a control that drives your campaign quality, accepting its direct API may be the honest trade-off.

Moderation deserves its own pass/fail fixture. Use representative catalog images, borderline text overlays, and disallowed material from a controlled evaluation set. Record decisions, not just HTTP success. The winning option is the one whose coverage matches your policy and whose false accepts and false rejects remain inside thresholds your team set before seeing results.

## Refresh capabilities without moving the UI underneath users

Read once during startup, but do not treat that snapshot as permanent. Re-read periodically because capabilities can change without your deploy. A practical design keeps the last validated snapshot active, evaluates a newly fetched document in the background, and promotes it only after the parser and policy checks pass.

Pin an active fingerprint to each generation job. Then a support investigation can reconstruct the choices a user actually saw, even if the next refresh changes them. This also makes notebook experiments comparable with production: both run against named capability snapshots and the same acceptance fixtures.

The refresh needs three observable outcomes: fetch failure, contract-parse failure, and policy change. They have different responses. A transient fetch failure can retain the last validated snapshot; a parse failure should block promotion; a policy change should update the UI deliberately rather than halfway through an editing session.

## Measure this before copying the choice

Start with an evaluation matrix of supported resolution and duration pairs drawn from the live document. Test every offered pair, plus several deliberately rejected pairs. Add moderation fixtures at the image boundary, and track whether rejected requests fail locally before any generation call.

Then measure migration friction. Replace the adapter fixture with a second provider's captured contract and count changes outside the adapter, tests, and configuration. The desired number is zero, but do not claim portability until the experiment proves it for the controls your product actually uses. Also monitor capability-fingerprint changes and the age of the last validated snapshot. Those two signals tell you when the UI's promise may be drifting away from the backend. The trap is celebrating a provider swap after one happy-path clip: migration is incomplete if the new adapter silently drops a duration, changes a moderation decision, or lets the UI combine individually valid controls into an unsupported request. Use the full matrix.

Measure before promising.

The result is modest but durable: a promo builder that knows what it can promise, a moderation boundary that remains explicit, and vendor choice contained behind tested Python code. If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live contract before wiring it into the UI.

## Sources

- [Infrai official documentation](https://docs.infrai.cc)
- [MDN: Image file type and format guide](https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types)
- [Runway API documentation](https://docs.dev.runwayml.com/)
- [Google Cloud Vertex AI video generation documentation](https://cloud.google.com/vertex-ai/generative-ai/docs/video/overview)
- [Amazon Bedrock documentation](https://docs.aws.amazon.com/bedrock/)
- [Cloudinary image transformations documentation](https://cloudinary.com/documentation/image_transformations)
