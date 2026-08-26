# One API Key for Multi-Provider Chat Moderation: A Python Structured Safety Classifier

Short answer: use one provider-neutral classifier contract, but keep the API key, model choice, and tenant cost ledger behind that contract. For a Python support-ticket triage system, the important result is a validated safety decision with an attributable cost, not a promise that OpenAI, Claude, and Gemini will behave identically.

That constraint changes the implementation. A notebook can print a confident paragraph and still be useless to a queue worker. The production path needs a small decision object, a versioned rubric, an explicit fallback policy, and enough usage metadata to answer “which tenant paid for this classification?” later. Keep the boundary boring. Boring boundaries survive provider changes.

## How should one API key route chat moderation with structured output?

Treat the unified chat-completions interface as a transport abstraction, not as a safety standard. The application sends the ticket text and a rubric; the classifier returns a closed set of actions such as `allow`, `review`, or `block`, plus categories, severity, and a reason. The surrounding code should validate that object before it touches a ticket queue.

The API key is an operational detail. Store it outside the repository, give the worker only the permission it needs, and attach a tenant identifier to every classification event. Do not use a shared key as a substitute for accounting. A gateway may receive requests for many customers, so the event record needs the tenant, request ID, rubric version, model label, input-token count, output-token count, action, and latency. If the upstream response does not provide usage data, record that absence explicitly instead of inventing a cost.

Keep it small.

This is the contract I would put beside the eval harness:

```python
from dataclasses import dataclass
from typing import Literal, Protocol


Action = Literal["allow", "review", "block"]


@dataclass(frozen=True)
class SafetyVerdict:
    action: Action
    categories: tuple[str, ...]
    severity: int
    reason: str


@dataclass(frozen=True)
class Usage:
    input_tokens: int | None
    output_tokens: int | None


@dataclass(frozen=True)
class Classification:
    verdict: SafetyVerdict
    usage: Usage
    model_label: str
    rubric_version: str


class ChatClassifier(Protocol):
    def classify(self, text: str) -> Classification:
        """Return a validated verdict and its accounting metadata."""
```

The provider adapter owns message formatting and structured-output options. The ticket worker owns neither. That separation lets an eval run compare two adapters with the same labeled input, while the queue code sees one result shape. It also makes a malformed response a visible adapter failure rather than a mysterious moderation decision.

## The failure modes that make a unified classifier unsafe

The first failure is schema confidence. A model can follow the policy in prose and still return a missing category, an unknown action, or a severity outside the range your database accepts. Parse and validate the response. Reject extra fields if the consumer has no defined meaning for them. A retry can address a transient transport problem; it cannot repair a rubric that produces inconsistent labels.

The second is taxonomy drift. “Harassment” may mean one thing to a support team and another thing to a model provider. Define categories in the product's language, include examples for borderline tickets, and version the rubric. When a policy owner changes an example, the eval set should change with it. Otherwise the dashboard compares unlike decisions while pretending nothing moved.

The third is tenant-blind cost. A single key makes credentials simpler, but it can hide a noisy tenant or a retry storm. Count each attempt separately, retain a correlation ID, and distinguish successful, rejected, and retried calls. Aggregate by tenant and by rubric version. The goal is not a perfect invoice replica; it is a defensible explanation of usage and a way to catch an accidental cost multiplier.

Here is a deliberately small ledger function. It does not assume a price, because prices and token accounting policies are external configuration and can change.

```python
from collections import defaultdict


def add_usage(
    totals: dict[str, dict[str, int]],
    tenant_id: str,
    usage: Usage,
) -> None:
    tenant = totals.setdefault(
        tenant_id,
        {"classified": 0, "input_tokens": 0, "output_tokens": 0},
    )
    tenant["classified"] += 1
    tenant["input_tokens"] += usage.input_tokens or 0
    tenant["output_tokens"] += usage.output_tokens or 0


tenant_totals: dict[str, dict[str, int]] = defaultdict(dict)
```

The example records counts and tokens, not a made-up dollar figure. In production, persist the event before acknowledging the ticket or use an idempotency key so a worker retry does not create a second billable interpretation in your own ledger.

The trade-off is straightforward: one credential and one contract reduce integration work, while the application takes on more responsibility for policy, accounting, and evaluation.

| Design | Best fit | Main limitation |
| --- | --- | --- |
| Provider-neutral gateway | Several approved chat backends behind one interface | Adds a routing boundary and does not standardize safety quality |
| Direct provider adapter | A team committed to one backend relationship | Provider-specific code grows when another backend is added |
| Dedicated moderation system | A queue with a matching fixed taxonomy | Less suitable when the product needs a portable custom rubric |
| Self-hosted classifier | Strict internal data and deployment requirements | The team owns serving, capacity, updates, and evaluation |

## A focused Python path from notebook to production

Start with a fixed fixture of support tickets: ordinary product questions, abusive language, requests that contain personal data, and ambiguous cases that should reach a person. The fixture is more useful than a generic benchmark because the labels describe the queue's actual policy. Split it by tenant characteristics when policy or language differs; otherwise a large, easy tenant can hide poor behavior for a small one.

Run the same rubric and schema through each approved adapter. Measure false allows and false blocks separately. Measure first-pass schema validity, review rate, latency, retries, and input/output tokens per ticket. A single accuracy score is too blunt: a false allow can carry a very different operational cost from a false block, and a cheap-looking prompt can become expensive when retries and review volume are included.

The harness can stay provider-neutral:

```python
def evaluate(
    classifier: ChatClassifier,
    cases: list[tuple[str, Action]],
) -> dict[str, float]:
    correct = 0
    for text, expected_action in cases:
        result = classifier.classify(text)
        if result.verdict.action == expected_action:
            correct += 1
    return {"action_accuracy": correct / len(cases)} if cases else {}
```

That function is intentionally incomplete as a safety score. Extend it with per-class confusion counts, schema failures, and a review of disagreements before promoting an adapter. Freeze the fixture, rubric version, and model label together. When one changes, rerun the gate. Notebook-to-prod means preserving the inputs and decisions that made the result acceptable, not merely copying the prompt into a service.

I’m not sure a public score can predict the cost of a support team's false-positive queue. Your mileage may vary. The uncertainty is resolved by labeling a representative sample, setting thresholds with the people who handle reviews, and measuring tenant-level usage after deployment.

## When should a different design win?

The one-key pattern is not suitable when a contractual or regulatory requirement calls for a purpose-built moderation control, when an organization cannot accept an intermediary, or when its rubric depends on private deployment and local data handling. Use a direct provider integration when the extra portability is not worth another routing boundary. Use a self-hosted classifier when the team can own serving, updates, capacity, and evaluation. Use a dedicated moderation system when its taxonomy matches the queue and the team does not want to maintain a prompted classifier.

The limitation is that portability stops at the interface. A common request shape cannot guarantee common refusal behavior, latency, or token usage, so a team that cannot run recurring evaluations should choose the simpler direct design.

For tickets containing protected health information, the API shape is not a compliance claim. Data handling, access controls, audit records, retention, and contractual obligations need their own review; 45 CFR Part 164 is a relevant primary reference for HIPAA Security and Privacy Rules. The design should make that review possible by keeping tenant identity, request identity, policy version, and decision history explicit.

The durable choice is therefore an ownership decision. A unified chat interface can reduce adapter code, but it does not own your taxonomy, evaluation policy, or tenant accounting. Keep those three in your application. Then a provider swap is a measured experiment, and the support queue remains understandable when the experiment fails.

## References

- [OpenAI Batch API guide](https://platform.openai.com/docs/guides/batch)
- [45 CFR Part 164, HIPAA Security and Privacy Rules](https://www.ecfr.gov/current/title-45/subtitle-A/subchapter-C/part-164)
