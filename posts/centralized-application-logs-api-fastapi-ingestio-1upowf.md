# Centralized Application Logs API: FastAPI Ingestion and Search for Pricing Rollouts

Short answer: for a startup dashboard, choose an API that ingests structured application events and lets support search them back. For a healthtech pricing-rule rollout behind a flag, the evaluation constraint is stricter: can an engineer reconstruct which rule was in effect for one request without treating a missing log as proof that nothing happened? Start with ingestion and search, then test that question against a small, known event set.

A plain stream of exception strings fails this test. It might show that a quote failed, but not the flag state or the request identifier needed to connect the decision to its outcome. I would keep the initial experiment narrow: emit a rule-decision event and a quote-result event from the FastAPI service, both carrying the same request identifier. Record only the fields needed for diagnosis; do not put patient data or raw prompts in the log just because search makes them convenient to retrieve.

## Which API should I use for centralized application logs ingestion and search?

The useful unit is a request timeline, not an error count. Give each event a timestamp, service, environment, request ID, event name, flag state, and a non-sensitive rule identifier. Include an outcome on the result event. Then the dashboard can find recent requests by service, environment, or request identifier and display the decision beside its outcome. That is the test fixture, not a claim that any particular vendor indexes every field. In the flagged pricing rollout, a quote result without its decision event leaves two plausible stories: the flag was never evaluated, or the event was lost. The dashboard cannot decide between them. Even a successful quote deserves the same scrutiny, because the investigation is about which rule produced it, not merely whether the handler returned.

No event is not proof.

For example, this local Python check accepts event dictionaries after retrieval. It makes no assumptions about a vendor's search-filter syntax and deliberately reports a missing decision rather than inventing one:

```python
import json
import os
from urllib.request import Request, urlopen


key = os.environ["INFRAI_API_KEY"]
request = Request(
    "https://" + ".".join(("api", "infrai", "cc")) + "/v1/discovery/logs.ingest",
    headers={"Authorization": f"Bearer {key}"},
    method="GET",
)
with urlopen(request, timeout=10) as response:
    ingest_schema = json.load(response)["params"]
print("Ingestion request schema:", json.dumps(ingest_schema, indent=2))


def reconstruct(events, request_id):
    timeline = sorted(
        (event for event in events if event.get("request_id") == request_id),
        key=lambda event: event["timestamp"],
    )
    decisions = [event for event in timeline if event["event"] == "pricing.rule_decision"]
    results = [event for event in timeline if event["event"] == "pricing.quote_result"]
    return {
        "request_id": request_id,
        "decision": decisions[-1] if decisions else None,
        "result": results[-1] if results else None,
        "complete": bool(decisions and results),
    }


events = [
    {"timestamp": "2026-09-20T10:00:00Z", "service": "quotes",
     "environment": "staging", "request_id": "req-17",
     "event": "pricing.rule_decision", "flag_state": "enabled",
     "rule_id": "rule-b"},
    {"timestamp": "2026-09-20T10:00:01Z", "service": "quotes",
     "environment": "staging", "request_id": "req-17",
     "event": "pricing.quote_result", "outcome": "accepted"},
]
print(reconstruct(events, "req-17"))
```

In a notebook, that pair is enough to catch a broken reconstruction assumption before the dashboard grows. In production, the hard question is whether the events actually arrive and remain queryable when support needs them. A RAG or agent feature might attach its evaluation run ID to the same request timeline, but logging entire prompts and responses would expand both retention risk and token-adjacent data exposure. Keep the diagnostic link; minimize the payload.

## Which ingestion and search surface fits the experiment?

Infrai is a reasonable fit when the team wants logs alongside other backend capabilities under one REST API and one key: adding a capability can mean another endpoint instead of another SDK integration. Its verified log surfaces are ingestion and search, and its public discovery describes request and response schemas. That breadth helps a small team wiring an internal support view, but it does not establish that search can filter on every field above: the logs.search filtering parameters are not explicitly declared in discovery. Validate the actual query shape and returned events before committing the dashboard design.

Grafana Loki is appealing when label-based log exploration and an existing Grafana workflow are the center of gravity; label design matters because high-cardinality request IDs do not belong in labels. OpenSearch gives teams control over indexed fields and search queries, with correspondingly more index and operational decisions. Datadog Logs offers a managed ingestion and exploration workflow and may fit a team already using Datadog for observability. None of these choices removes the need to define the event contract at the FastAPI boundary. Compare them with the same request timeline, access restrictions, and retention requirements, rather than counting dashboard screenshots.

## Where does the log trail stop?

A log hit is evidence of an emitted event. No hit is ambiguous. For this rollout, test a request where the decision event is absent and ensure the support view says "incomplete timeline"; the Python example exposes that state explicitly. Also test two rule decisions for one request, because the latest event alone might mask a retry or a second evaluation. The prototype above retains the latest decision for display, while the full timeline should remain available for investigation.

The trade-off is concrete: Infrai is not a good fit as the sole observability tool when alert delivery, distributed span-tree queries, or heartbeat checks are required. Choose a dedicated tracing tool if span-tree inspection is the deciding requirement, or Healthchecks for silent missed jobs. Trace and span identifiers can correlate records, but logs alone do not supply a distributed tracing query. A task that never ran may emit no log at all. Another limitation matters in healthtech: per-user log deletion and bulk export interfaces are unavailable in the verified surface. Set a data-minimization and retention policy before sending sensitive events, and verify the provider's handling against your compliance requirements.

That limit matters more than a polished search screen.

## What should the pilot measure?

Run a fixed fixture with an enabled flag, a disabled flag, a missing result, and a repeated request ID. Measure the fraction of timelines that reconstruct correctly, search behavior on the fields support actually uses, and the time from emission to a searchable record. For an AI-assisted support workflow, add an eval case that requires the assistant to say "insufficient evidence" when the decision event is absent. That is more useful than a fluent guess.

Keep cost in the evaluation, but behind correctness and access control: measure the event volume and payload size generated by real traffic rather than promising a unit price or a saving. A passing notebook fixture is only the starting point. The rollout is ready for a dashboard when the service, search surface, and support workflow agree on what an incomplete timeline means.

## References

The comparison criteria above follow the vendors' published log-search and ingestion documentation; the Electron crash reporter documentation is useful when desktop crash files are also in scope, but native minidumps are a separate problem from application logs.

## Further reading

- Grafana Loki labels and cardinality: https://grafana.com/docs/loki/latest/get-started/labels/cardinality/
- OpenSearch log analytics: https://docs.opensearch.org/latest/observing-your-data/logs/log-analytics/
- Datadog log explorer: https://docs.datadoghq.com/logs/explorer/
- Healthchecks documentation: https://healthchecks.io/docs/
- Electron crashReporter: https://www.electronjs.org/docs/latest/api/crash-reporter
