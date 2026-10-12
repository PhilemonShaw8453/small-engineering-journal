# One API Key Changes Explained: FastAPI Access Reviews Across Capabilities

Short answer: make the single provisioning call create one account identity, an explicit set of capability grants, a spend ceiling, and evidence for the next access review. The returned credential should identify the account; it should not silently mean "everything allowed." For a fintech team, that distinction turns convenient onboarding into an access review someone can actually sign.

The least complex useful design is a small control-plane endpoint backed by an authoritative grant record. Runtime services read that record before accepting work. If projected usage would cross the account ceiling, they refuse the request and record why. One call reduces onboarding coordination, but it also concentrates mistakes: an overbroad default, a duplicate request, or a missing audit event now affects every capability at once.

## What actually changes when one call provisions every capability?

The unit of provisioning changes. Instead of creating separate credentials for ledger export, transaction classification, and case summarization, the platform creates one account and attaches narrow grants to it. That removes credential sprawl from the caller's workflow. It does not remove authorization, budgeting, rotation, revocation, or review work; those controls move behind a common control-plane boundary.

The data flow is short. An onboarding service sends an idempotency key, requested capabilities, and a monthly spend ceiling. The provisioning service validates the request, stores the account and grants in one transaction, creates a credential through a secrets-management boundary, and emits a durable review event. At runtime, a capability service resolves the credential to an account, checks the named grant, reserves room below the ceiling, and either accepts or refuses the operation. The review reads those same records rather than reconstructing intent from scattered service logs.

This is a control-plane simplification, not a universal runtime pass.

## A minimal FastAPI control plane

The example keeps the credential out of application logs, requires an idempotency key, rejects unknown capability names, and returns the secret once. Its in-memory stores make the sample runnable, not production-ready. A deployed version needs durable transactional storage, authenticated callers, encryption, rotation, and an audit sink whose retention matches the review policy.

```python
from datetime import datetime, timezone
from hashlib import sha256
from secrets import token_urlsafe
from threading import Lock
from typing import Literal
from uuid import uuid4

from fastapi import FastAPI, Header, HTTPException, status
from pydantic import BaseModel, Field

app = FastAPI()
lock = Lock()
accounts: dict[str, dict] = {}
idempotency: dict[str, dict] = {}
audit_events: list[dict] = []

Capability = Literal[
    "ledger_export",
    "transaction_classification",
    "case_summarization",
]


class ProvisionRequest(BaseModel):
    legal_entity_id: str = Field(min_length=3, max_length=80)
    capabilities: set[Capability] = Field(min_length=1)
    monthly_spend_ceiling_cents: int = Field(ge=10_000, le=10_000_000)
    reviewer_group: str = Field(min_length=3, max_length=80)


class ProvisionResponse(BaseModel):
    account_id: str
    api_key: str
    capabilities: list[Capability]
    monthly_spend_ceiling_cents: int


def fingerprint(api_key: str) -> str:
    return sha256(api_key.encode("utf-8")).hexdigest()


@app.post(
    "/accounts",
    response_model=ProvisionResponse,
    status_code=status.HTTP_201_CREATED,
)
def provision_account(
    request: ProvisionRequest,
    idempotency_key: str = Header(min_length=16, max_length=128),
) -> ProvisionResponse:
    request_shape = request.model_dump(mode="json")

    with lock:
        prior = idempotency.get(idempotency_key)
        if prior:
            if prior["request"] != request_shape:
                raise HTTPException(
                    status_code=status.HTTP_409_CONFLICT,
                    detail="Idempotency key was already used for another request",
                )
            return ProvisionResponse(**prior["response"])

        account_id = str(uuid4())
        api_key = f"acct_{token_urlsafe(32)}"
        capabilities = sorted(request.capabilities)
        created_at = datetime.now(timezone.utc).isoformat()
        accounts[account_id] = {
            "legal_entity_id": request.legal_entity_id,
            "capabilities": capabilities,
            "monthly_spend_ceiling_cents": request.monthly_spend_ceiling_cents,
            "reviewer_group": request.reviewer_group,
            "credential_fingerprint": fingerprint(api_key),
            "credential_status": "active",
            "created_at": created_at,
        }
        response = {
            "account_id": account_id,
            "api_key": api_key,
            "capabilities": capabilities,
            "monthly_spend_ceiling_cents": request.monthly_spend_ceiling_cents,
        }
        idempotency[idempotency_key] = {
            "request": request_shape,
            "response": response,
        }
        audit_events.append({
            "event": "account.provisioned",
            "account_id": account_id,
            "capabilities": capabilities,
            "spend_ceiling_cents": request.monthly_spend_ceiling_cents,
            "reviewer_group": request.reviewer_group,
            "occurred_at": created_at,
        })

    return ProvisionResponse(**response)
```

The caller cannot submit arbitrary capability strings. That makes typos fail closed and gives reviewers a finite vocabulary. It also keeps the provisioning contract separate from prompts, models, or downstream implementation details, which can change without rewriting the access decision.

The API key is shown once because the example needs to demonstrate delivery. Do not store the cleartext key in the account row or audit event. OWASP's secrets-management guidance recommends centralizing secrets, applying least privilege, automating rotation where possible, and auditing access to secrets. Those concerns remain even when the caller sees only one key.

## Make refusal an explicit product decision

A spend ceiling without a refusal rule is an annotation. Before launch, decide what happens when the next operation could cross the ceiling: reject it, queue it for approval, or permit a tightly bounded overage. For an access-review workflow, rejection is the easiest behavior to explain and verify. It is also the most visible to users.

There is no honest universal threshold. Start from the amount the accountable owner is authorized to approve, then evaluate the resulting refused traffic with replay data. The useful metric pair is accepted legitimate requests and refused legitimate requests under the same ceiling. A lower ceiling may reduce exposure while interrupting month-end review work; a higher ceiling may preserve flow while exceeding the amount the reviewer believed they approved. **The decision record needs both sides of that trade-off.**

For AI-backed capabilities, reserve budget before dispatch rather than adding estimated cost after completion. Reconcile the reservation when actual usage becomes available. Otherwise, concurrent requests can each observe the same remaining amount and collectively pass a ceiling they would have crossed in sequence. Use integer minor currency units rather than floating-point currency values.

Refusal events need enough context to debug policy, but never the credential or sensitive transaction content. Record the account ID, capability, policy version, requested reservation, remaining allowance, decision, timestamp, and a correlation ID. Then build an eval fixture with boundary cases: one unit below the ceiling, exactly at it, one unit above it, two concurrent reservations, a revoked grant, and a replayed provisioning request. The examples that shaped the policy become regression tests instead of a forgotten notebook.

Keep the eval small. Run it often.

## The review should reconstruct intent and reality

A reviewer needs a compact comparison, not a dump of every request. Put approved intent beside observed use and the proposed next action. A useful review row contains the legal entity, capability, approving group, grant date, last-used time, usage for the review window, refused-request count, ceiling, policy version, and recommendation to retain, narrow, or revoke.

That structure exposes quiet mismatches. A granted capability with no observed use may be removable. Repeated refusals near a known reporting deadline may indicate that the ceiling or workload schedule is wrong, not that the caller is hostile. Activity on a capability absent from the approved grant set is a control failure and should never be normalized as ordinary usage.

Consider a hypothetical month-end review with three grants. Ledger export is approved and used, transaction classification is approved but unused, and case summarization is approved yet repeatedly refused at the ceiling. Those rows should not produce one blanket "retain" decision merely because they share a key. The first row supports retention if the owner still needs the export. The second asks for a narrower grant or an explanation of upcoming use. The third demands a joint policy and workload check: raising the ceiling without examining request size could hide waste, while revoking the grant could block a legitimate reporting job. Now add a fourth row: an attempted capability that never appeared in the grant set. That is not ordinary refused traffic. It is evidence that enforcement worked and a reason to inspect the caller's configuration. This small dataset is also an eval fixture. Feed the same four cases through every policy revision and compare decisions before deployment, including the exact-at-ceiling boundary and two reservations arriving together. The point is not to maximize acceptance. It is to keep each refusal explainable to the reviewer who owns the risk and the team whose work was stopped.

**One credential must still yield capability-level evidence.** If all runtime events say only "this key was used," the unified key has erased the distinctions the reviewer must approve. Bind each accepted or refused operation to a named capability and the policy version that made the decision. Preserve grant history so a later review can tell what was allowed at the time, even after the current configuration changes.

The reviewer should not have to infer ownership. Make the accountable group and next-review date part of the grant. The example captures a reviewer group for that reason, although a real service should resolve it to a controlled directory identity rather than trusting decorative free text.

## Failure modes worth testing before rollout

The dangerous failures sit around the happy path. Retrying after a network timeout can create a second account unless provisioning is idempotent. Reusing the same idempotency key with a different payload must conflict; returning the first result would make the caller believe a changed request was applied. Partial writes can create a live credential without grants or grants without review evidence, so durable implementations should commit account state, grant state, and the durable audit handoff as one recoverable operation.

Revocation is another sharp edge. Disabling one capability should not require rotating the shared credential, while a suspected credential disclosure should disable the credential across every capability. Those are different actions with different blast radii. Test both. Also test stale authorization caches: a revocation that exists only in the control-plane database is ineffective until runtime enforcement observes it.

Prompt cost deserves separate attention because capability access and spend authorization are related but distinct. A caller may be permitted to use case summarization yet request inputs that consume the remaining allowance too quickly. Enforce a per-operation reservation and the account ceiling, then measure refusal outcomes in the eval harness. Avoid hiding this behind a provider-specific model name; the access review should survive a model change.

Rotation needs a transition. A practical sequence issues a replacement, permits a short overlap, observes adoption by credential fingerprint, and revokes the old credential at a declared deadline. The review log should show who initiated the rotation and when each state changed. Never log the secrets themselves.

No guesswork.

## Operational acceptance, in prose

Before enabling the endpoint, walk one synthetic fintech account from request through review. Submit the same provisioning request twice and confirm there is still one account. Change the payload under the same idempotency key and expect a conflict. Exercise every granted capability, attempt one capability that was not granted, and cross the spend boundary with concurrent reservations. Rotate the credential, revoke a single capability, then revoke the account credential. The resulting review should explain every acceptance and refusal without exposing the key or transaction content.

Ship only when the on-call engineer can distinguish policy refusal from dependency failure, the security owner can revoke access promptly, and the finance owner can trace the ceiling back to an approval. Track refusal rate by capability and policy version, but pair it with accepted volume and scheduled workload context. A raw refusal count invites the wrong fix.

The final design rule is compact: provision identity, grants, budget, and review evidence together; enforce them separately at runtime. **The single call succeeds when it reduces onboarding steps without collapsing distinct controls into one opaque key.**

## Further reading

- OWASP Secrets Management Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
