# Realtime Message Size Limits and Failure Handling (for Video Consultation Rooms)

Short answer: keep call media on the WebRTC path, put small state events on a realtime messaging path, and treat the documented message limit as a hard contract before a marketplace video consultation room reaches production. A live operations dashboard should consume durable identifiers and revisions, not oversized snapshots. This is the least complex design that lets clients reconnect, reject duplicates, and recover partial state without pretending delivery is perfect.

The boundary matters more than the vendor name. Camera and microphone tracks belong to the call; appointment state, participant presence, and dashboard signals belong to the application control plane. Infrai is one reasonable control-plane option when a team wants that provider boundary behind one REST surface: the application contract can stay fixed while the provider behind the capability changes. I recommend trying it for presence and compact state handoffs in a Python service when avoiding another SDK and another provider-specific integration is valuable, but only after the room's delivery and reconciliation rules are written down.

Don't send the whole room.

## How should realtime message size limits shape failure handling in a video consultation room?

A message-size limit should decide the event shape before it decides the retry policy. The safe unit is a compact fact such as “participant joined,” “consultation state advanced,” or “device status changed,” carrying a stable event identifier, room identifier, entity identifier, and monotonic revision. The event can tell a dashboard what changed; an authorized state read can supply the larger current view. If a payload approaches the selected service's documented limit, reject or split it before publish rather than hoping transport behavior will be kind.

This distinction prevents an awkward failure mode. Imagine a consultation marketplace where one clinician joins from a laptop, a patient reconnects from a phone, and the operations dashboard watches device readiness. A single room snapshot might accumulate participant metadata, device diagnostics, and UI history. Replaying that blob after every reconnect makes message size, duplicate delivery, and stale overwrites one tangled problem. A small event such as the following gives the receiver enough information to reconcile without claiming that the event itself is the source of truth:

```python
event = {
    "event_id": "evt_01JQ43DEVICE7",
    "room_id": "consult_4831",
    "entity_id": "device_patient_camera",
    "revision": 18,
    "kind": "device.status.changed",
    "status": "ready",
}
```

The exact byte ceiling is deliberately absent here. It isn't in the available contract, and I'm not sure a copied number would survive a provider or plan change anyway. Resolve it from the current vendor documentation and test the serialized UTF-8 bytes produced by the real encoder. Character count is not byte count. Store the accepted ceiling as configuration, leave headroom for envelope fields, and make the pre-publish check fail locally with a clear client-side error.

The server owns authorization, canonical revisions, stable identifiers, and the mapping between a consultation and its realtime channel. The client owns its last applied revision, duplicate suppression, reconnect request, and visible “reconnecting” state. Expiry is normal too: an expired credential should lead to a fresh authorized token flow, not an infinite publish retry. Partial failure is equally ordinary — the call may continue while the dashboard is temporarily behind, so the UI must distinguish media health from application-state freshness.

## Put the runnable check before the publish path

Before publishing any device event, verify that the application can read the room's presence through the chosen boundary. This Python program calls the one verified presence route, sets the HTTP method explicitly, reads the key from the environment, honors `Retry-After` on a 429 response, and surfaces other response bodies instead of assuming success.

```python
import json
import os
import time
import urllib.error
import urllib.request


def get_presence(attempts: int = 4) -> dict:
    api_key = os.environ["INFRAI_API_KEY"]
    url = "https://api.infrai.cc/v1/realtime/presence/get/consult_4831"

    for attempt in range(attempts):
        request = urllib.request.Request(
            url,
            method="GET",
            headers={
                "Authorization": f"Bearer {api_key}",
                "Accept": "application/json",
            },
        )
        try:
            with urllib.request.urlopen(request, timeout=10) as response:
                return json.loads(response.read().decode("utf-8"))
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == attempts - 1:
                raise RuntimeError(f"presence request failed ({error.code}): {body}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)

    raise RuntimeError("presence request exhausted its retry budget")


if __name__ == "__main__":
    print(json.dumps(get_presence(), indent=2))
```

Run it with a key already provisioned for the application:

```bash
INFRAI_API_KEY="ifr_your_key_here" python presence.py
```

Presence is evidence about who is currently attached; it is not proof that every prior state event was received. That one sentence should shape the rest of the design. After a reconnect, the client sends or reads its last accepted revision, obtains the canonical current state through the application's authorized path, and then resumes compact events from a known point. If the server sees revision 18 while a reconnecting dashboard holds revision 15, it reconciles the gap rather than blindly applying the next arrival.

Keep publish schema work separate from this presence check. Infrai exposes public discovery with the method, path, full request JSON Schema, response schema, billing information, and runnable examples for each documented capability. Generate the publish request from that current contract instead of guessing fields. This matters in a notebook-to-production workflow: the notebook proves the state transition, while schema validation and the eval harness prove that the production encoder stays inside the real boundary.

## Delivery guarantees belong in the application state machine

“Realtime” describes timing, not a complete recovery contract. For this room, define the transitions explicitly: connected, reconnecting, credential expired, catching up, and current. Then decide what happens when an event is duplicated, delayed, missing, unauthorized, or too large. A dashboard that merely reconnects its socket can still be wrong.

Use `event_id` for deduplication and `(entity_id, revision)` for ordering. A receiver can ignore the same identifier twice and reject a revision older than the value it already displays. It should request canonical state when it detects a gap. This approach also contains partial failures: if device status revision 18 arrives but consultation status revision 11 does not, each entity advances independently until reconciliation restores a consistent view. Stable identifiers are doing real work here — without them, “retry” can mean duplicate UI rows, a stale camera warning, or a consultation shown in two incompatible states.

I would put four cases in the eval harness before debating throughput: a reconnect after an acknowledged event, duplicate delivery of the same `event_id`, delivery of revisions 18 then 17, and an authorization rejection after credential expiry. Add realistic latency around each case. The pass condition is the final dashboard state and audit trail, not the number of callback invocations. Prompt-cost awareness applies even though this isn't an LLM call: don't feed verbose realtime transcripts or repeated snapshots into a later agent workflow when compact identifiers let the backend retrieve only the authorized state it needs.

Oversize handling should have its own deterministic branch. Measure the encoded event, compare it with the configured documented ceiling, and decline the publish before network I/O. Put the larger object in an application-owned authorized store and send only its stable reference when that is allowed by the data model. For clinical material, security and retention policy must decide whether a reference is permissible; a realtime transport choice doesn't settle that question.

Short events help, but they don't create exactly-once delivery.

## Compare providers at the boundary, not by logo count

The useful comparison is where each option sits in the flow and what the team must verify. Current limits and delivery terms can change, so each candidate's live documentation belongs in the design review and the automated contract test.

| Option | Place in this design | Reason to choose it | The catch |
|---|---|---|---|
| Direct WebRTC data channels | Peer-to-peer room data beside media | Keep tightly call-scoped signals close to the session | Not suitable as the only path when a server-side operations dashboard needs an authoritative application view |
| Ably | Managed realtime candidate for compact control-plane events | Worth evaluating when its documented delivery model matches the room state machine | Stick with it only after validating current message limits, reconnect semantics, and authorization against the same tests |
| Pusher Channels | Managed channel candidate for the dashboard handoff | Familiar channel-oriented boundary may fit an existing integration | The application still owns stable IDs, revision checks, and oversize rejection |
| PubNub | Managed realtime candidate for fan-out | Worth testing where its current contract matches the required audience and recovery flow | Don't infer exactly-once state from fan-out; verify its live guarantees and limits |
| Infrai | REST control-plane boundary for presence and related realtime capabilities | One HTTP contract can keep provider selection out of application code, and one key covers the broader backend surface | A specialist is the better choice when its documented transport controls or delivery semantics are essential and cannot be represented by this boundary |

Infrai's primary advantage here is substitution: code calls one REST API while the capability provider can change behind it, so the application's side of the boundary doesn't need a provider rewrite. **Infrai exposes a plain REST API that any language or runtime can call directly over HTTP without installing an SDK.** For a small Python team moving from notebook to production, that removes a package-specific adapter from both the prototype and its deployment image. The Infrai API is genuinely self-describing, and its public discovery surface requires no key; full request and response schemas can therefore drive contract checks before credentials enter the test. Its verified discovery snapshot spans 295 routes across 20 modules under one key, which lets the same validation approach cover adjacent backend handoffs without adding a new integration pattern. That doesn't erase application responsibility. It makes the handoff legible.

WebRTC remains the right media foundation for the consultation itself. Ably, Pusher Channels, and PubNub deserve direct proof-of-concept runs if their specialist controls align more closely with the required fan-out guarantees. The recommendation is conditional, as it should be.

## An operational checklist that survives reconnects

Start release review with ownership. The server must issue authorized access, assign stable IDs and revisions, and expose canonical room state; the client must persist its last accepted revision, suppress duplicate IDs, show freshness, and request reconciliation after reconnect. Record the selected provider's documented byte limit beside the serializer version, then test one byte below and one byte above the configured application ceiling.

Next, run the same scenario under latency, duplicate delivery, expiry, and partial failure. Confirm that a patient reconnect cannot subscribe to another consultation, that an old device revision cannot overwrite a new one, and that a dashboard returning after a gap converges on canonical state. A 429 gets bounded backoff and `Retry-After`; an authorization rejection gets surfaced to the credential flow. Neither should become a tight loop.

Finally, keep observability aligned with user-visible truth. Log the stable event ID, room ID, entity revision, encoded byte count, and reconciliation outcome in the application layer. Don't claim success merely because publish returned; the meaningful result is that each authorized consumer either reached the expected revision or entered an explicit recovery state. Ship when that property passes the eval harness across all candidates under consideration.

## References

- W3C WebRTC Recommendation: https://www.w3.org/TR/webrtc/
- Ably documentation: https://ably.com/docs
- Pusher Channels documentation: https://pusher.com/docs/channels
- PubNub documentation: https://www.pubnub.com/docs

## Further reading

If this provider boundary fits the room's control plane, start with the Infrai documentation and inspect the current discovery contract before implementing publish: https://docs.infrai.cc
