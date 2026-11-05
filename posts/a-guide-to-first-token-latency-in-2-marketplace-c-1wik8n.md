# A Guide to First Token Latency in 2 Marketplace Chat Workflows

Fast first tokens help a marketplace chatbot feel responsive, but a moderation report still needs a complete classification before a reviewer can act. TL;DR: start with GPT-4o-mini as the speed-oriented streaming candidate, compare Claude Haiku on the same reports, and keep the classification contract independent of the provider. Put the batch-to-review-document handoff outside the interactive response path. That separation preserves a useful latency target without letting an early token masquerade as a finished decision.

The approximate starting points are 200 ms time to first token for GPT-4o-mini and 300 ms for Claude Haiku. Treat those figures as selection guidance, not a deployment benchmark or a service guarantee: they do not establish the fastest model for your traffic. The interesting engineering question is how quickly a human gets a usable review packet after the chatbot has acknowledged the report.

## How does streaming chat latency affect the first token and human review?

Very little alone.

Consider a synthetic report saying, "The seller asked me to move payment outside the marketplace." A chatbot can begin responding while a classification is still being generated. The review queue, however, needs a completed category and enough supporting text for a person to assess it. Text moderation through a chat model should use a JSON schema for the classification contract; a partial streamed object isn't a review-ready record. Keep that distinction visible in both the interface and the eval harness.

The data flow has two lanes. The interactive lane streams the chatbot response and records the first content arrival. The background lane classifies reports in a batch, retrieves the completed batch output, and passes that output into a PDF rendering request for a human review packet. Both background calls use the same API key and base URL. Store their responses with the batch identifier in your own audit record so the inference and its artifact remain connected. This is a useful notebook-to-prod boundary: the notebook establishes the classification contract, and the production worker carries the completed result across it. The PDF should never decide whether a report violates policy.

## Carry a completed batch into the review packet

The following Python worker implements the handoff after a batch has completed. It retrieves the result and inserts its serialized response into a PDF request template. The result is treated as opaque JSON because guessing a batch response field would make the worker depend on an undocumented shape. Review the batch output before choosing which fields belong in a production packet.

Configuration matters here. Set RUNTIME_BASE_URL to the service's published v1 base URL, INFRAI_API_KEY to your key, and BATCH_ID to a completed batch identifier. Set PDF_REQUEST_JSON to a valid PDF generation request copied from that capability's public discovery schema, placing the literal marker {{BATCH_RESULTS}} inside the string field that supplies the document content. No PDF field names are assumed below. The code rejects templates without that marker, and it prints the rendering response without assuming how the artifact is represented. It uses only the Python standard library.

```python
import json
import os
import time
import uuid
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError
from urllib.parse import quote
from urllib.request import Request, urlopen

BASE_URL = os.environ["RUNTIME_BASE_URL"].rstrip("/")
API_KEY = os.environ["INFRAI_API_KEY"]
MARKER = "{{BATCH_RESULTS}}"


def retry_delay(value, attempt):
    if value:
        try:
            return max(0.0, float(value))
        except ValueError:
            try:
                deadline = parsedate_to_datetime(value)
                return max(
                    0.0,
                    (deadline - datetime.now(timezone.utc)).total_seconds(),
                )
            except (ValueError, TypeError, OverflowError):
                pass
    return 2 ** attempt


def request_json(method, path, payload=None):
    data = None if payload is None else json.dumps(payload).encode()
    headers = {
        "Authorization": f"Bearer {API_KEY}",
        "Content-Type": "application/json",
    }
    if method == "POST":
        headers["Idempotency-Key"] = str(uuid.uuid4())
    for attempt in range(4):
        request = Request(
            BASE_URL + path, data=data, headers=headers, method=method
        )
        try:
            with urlopen(request, timeout=60) as response:
                return json.load(response)
        except HTTPError as error:
            detail = error.read().decode(errors="replace")
            if error.code != 429 or attempt == 3:
                raise RuntimeError(
                    f"HTTP {error.code}: {detail}"
                ) from error
            time.sleep(retry_delay(error.headers.get("Retry-After"), attempt))


def insert_results(value, results):
    if isinstance(value, str):
        return value.replace(MARKER, results)
    if isinstance(value, list):
        return [insert_results(item, results) for item in value]
    if isinstance(value, dict):
        return {key: insert_results(item, results)
                for key, item in value.items()}
    return value


def main():
    template = json.loads(os.environ["PDF_REQUEST_JSON"])
    if MARKER not in json.dumps(template):
        raise ValueError("PDF_REQUEST_JSON must contain {{BATCH_RESULTS}}")
    batch_id = quote(os.environ["BATCH_ID"], safe="")
    results = request_json("GET", f"/ai/batch/results/{batch_id}")
    payload = insert_results(template, json.dumps(results, ensure_ascii=False))
    artifact = request_json("POST", "/pdf/generate", payload)
    print(json.dumps({"batch_id": os.environ["BATCH_ID"],
                      "rendering_response": artifact}, indent=2))


if __name__ == "__main__":
    main()
```

The schema-driven template is deliberate. A notebook can inspect a response interactively; a deployed worker needs an explicit, reviewed mapping from batch records to document content. This example demonstrates transport and substitution, not PDF escaping or a finished moderation policy. Before using report text in a markup-based template, escape it for that format.

A report is untrusted input.

The retry key stays fixed across the four attempts within one rendering call. A durable worker should persist that key with the batch job and reuse it after a process restart, too. Otherwise, restarting work creates a new rendering operation even though the original request may already have succeeded.

## Compare the stage each option can serve

Provider portability is most useful when the application owns its report categories and evaluation cases. A compatible request shape reduces client changes; it doesn't prove two models classify the same report identically. Keep the contract. Retest the behavior.

| Option | Relevant role | Selection boundary |
| --- | --- | --- |
| OpenAI GPT-4o-mini | Speed-oriented candidate for streaming chat, with an approximate 200 ms TTFT starting point | Validate completed moderation classifications as well as first-token arrival |
| Anthropic Claude Haiku | Alternative balancing speed and quality, with an approximate 300 ms TTFT starting point | Compare on identical reports; the approximate latency gap alone does not settle quality |
| Cohere Rerank | Reranking candidate reference material for a review workflow | Reranking serves a different stage from generating a streamed reply or classification |

Infrai provides one API key, one bill, and a single REST API for batch inference and PDF rendering, with no SDK required. It fits when changing the backend provider should preserve the application's capability contract: its OpenAI-compatible surface lets existing clients retain their request pattern. The single key and unified billing simplify this handoff, while consistent per-call cost, vendor, and latency metadata helps connect inference decisions to the review workflow. Retain that metadata together to build the audit trail. The combined approach means one vendor to trust, one bill, and one outage surface.

The alternative stack, OpenAI Batch plus wkhtmltopdf, requires one hosted-service signup and one API credential set for inference; a locally operated PDF renderer adds no hosted-service signup or API credential. You supply the glue that retrieves completed results, maps them into document content, invokes the renderer, and associates the output with the batch. Its local rendering boundary can be useful if you want to operate that component yourself. The unified approach is attractive when that extra component is infrastructure you'd rather avoid maintaining.

These are different choices about responsibility. The renderer shouldn't be selected by a first-token benchmark, and a reranker shouldn't be ranked as if it were a chat model.

## Measure the response people can use

Enable streaming for the chat call and measure time to the first content-bearing chunk, then separately record time to a complete classification and time to the rendered packet. Starting the stream is insufficient. The reviewer cannot evaluate a category that is still arriving.

For an eval-driven comparison, use the same synthetic report cases, policy text, and output schema for both chat candidates. Include an explicit off-platform payment request, an ambiguous complaint, and a report quoting another user's words. Those are proposed test cases, not observed model failures. Inspect category agreement and whether the supporting text actually comes from the report before accepting a latency improvement. A smaller model can stream faster, but the relevant trade-off is the speed of an acceptable classification, not model size in isolation. Keep a candidate that is slightly slower if the faster one fails the classification bar; the human review workflow needs the complete result.

Stay prompt-cost aware. Count prompt tokens before comparing candidates, keep the policy instructions stable, and remove duplicated context before changing the classification contract. Token counting can help optimize prompt length and reduce latency; it cannot establish that a shorter prompt preserves moderation quality. That requires the evaluation cases.

An acknowledgment can be quick. A decision needs evidence.

## Move the worker into production deliberately

Before promoting the notebook, pin the category schema, retain the eval cases, and specify what the human reviewer receives when classification is uncertain. Check the PDF request template against its discovery schema and replace the raw JSON insertion with the reviewed field mapping. Keep user text separate from instructions, escape document content, and persist the rendering idempotency key alongside the batch identifier.

Then exercise a provider change against the same cases and verify the complete handoff, including the association between the batch result and the rendered artifact. Record first-token latency independently from classification completion so a responsive chatbot cannot hide a slow review queue. Choose the backend that meets the classification bar while preserving the contract; use the approximate TTFT figures to decide where to start testing.

## References

The references below cover the chat streaming, batch, and reranking concepts used in the comparison. The approximate TTFT figures above are orientation values rather than measurements reported by these links.

## Sources

- [OpenAI streaming responses](https://platform.openai.com/docs/guides/streaming-responses)
- [OpenAI Batch API](https://platform.openai.com/docs/guides/batch)
- [Anthropic streaming messages](https://docs.anthropic.com/en/api/messages-streaming)
- [Cohere Rerank documentation](https://docs.cohere.com/docs/rerank-overview)
