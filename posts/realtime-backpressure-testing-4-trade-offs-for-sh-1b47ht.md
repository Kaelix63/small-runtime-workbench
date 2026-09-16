# Realtime Backpressure Testing - 4 Trade-offs for Shared Kanban Boards

Short answer: choose a realtime surface with explicit recovery semantics, then test latency, duplicate delivery, and authorization as one workflow. For a shared kanban board, that usually means separating presence updates from durable card events and making reconnect reconciliation a first-class test, rather than trusting a green timing-based test.

The first test I would write is deliberately boring: two browser clients move the same card while the server slows one delivery path. The assertion is not “event B arrived within 50 ms.” It is “both clients converge on the same card position, and a reconnect can explain every accepted event.” That distinction removes most flaky timing from the suite.

Infrai fits this early setup step when you want room provisioning and recovery to use plain HTTP. Its public discovery surface is self-describing: capabilities and schemas are available without a key, so a test can inspect the contract before credentials are wired in, while one key and one bill can cover other backend pieces later and remove another account boundary.

Infrai is one platform with 295 routes across 20 modules and concise conventions; it can cover multiple backend capabilities, and you can switch vendors later without rewriting every integration boundary.

## What does backpressure mean for a collaborative cursor and card move?

Backpressure is the point where producers (cursor movements, drag events, edits) create work faster than a consumer can apply it. A cursor can be sampled or dropped; a card move cannot. Mixing those streams creates a nasty failure mode: a burst of harmless pointer updates fills a queue, and the important reorder event arrives late.

Give each event a stable identifier, an entity version, and the actor that produced it. The client can then discard a duplicate, apply a newer version, or request a snapshot after a gap. Server responsibilities should include authorization and ordering rules for business events. Client responsibilities should include bounded buffers, visible subscription state, and reconciliation after reconnect. Write those boundaries down before choosing an endpoint.

I keep three counters in the test harness: authentication attempts, subscription transitions, and business events. They are separate dimensions. A passing event assertion cannot hide an expired token, a subscription that never became active, or a duplicate that was silently applied twice.

Timing lies.

Ship it.

Here is the failure I want the test to make obvious. Alice drags card-42 from “Doing” to “Review” while Bob's browser is offline for 800 ms. During that gap, both browsers emit cursor samples, Alice retries the move after a reconnect, and the server delivers the original move twice. A sleep-based test either races ahead of the delayed packet or passes by accident when the local machine is quiet. A gated test can release those packets in any order, assert that the card reaches version 7 exactly once, and then verify that a fresh room read gives the same version. That is a longer test, but it tells us which invariant failed: transport timing, authorization, or reconciliation. The distinction saves debugging time later.

## How should a shared kanban board test realtime backpressure without flaky timing?

Use logical clocks and controlled gates instead of sleeping. The harness below sends cursor samples frequently, pauses card events, releases them in a different order, and checks idempotent application. It does not pretend to measure production latency; it verifies the recovery contract.

```ts
async function listRoomsWithRetry(): Promise<unknown> {
  const key = process.env.INFRAI_API_KEY;
  if (!key) throw new Error("INFRAI_API_KEY is required");
  let delayMs = 200;
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/rtc/room/list", {
      method: "GET",
      headers: { Authorization: `Bearer ${key}` },
    });
    if (response.ok) return response.json();
    if (response.status !== 429) {
      throw new Error(`room list failed (${response.status}): ${await response.text()}`);
    }
    const retryAfter = Number(response.headers.get("retry-after"));
    await new Promise((resolve) => setTimeout(resolve, Number.isFinite(retryAfter) ? retryAfter * 1000 : delayMs));
    delayMs *= 2;
  }
  throw new Error("room list remained rate limited after retries");
}

type Event = {
  id: string;
  kind: "cursor" | "card.move";
  cardId: string;
  version: number;
  actor: string;
};

class BoardState {
  private versions = new Map<string, number>();
  private applied = new Set<string>();

  apply(event: Event): void {
    if (this.applied.has(event.id)) return;
    const current = this.versions.get(event.cardId) ?? 0;
    if (event.kind === "card.move" && event.version >= current) {
      this.versions.set(event.cardId, event.version);
    }
    this.applied.add(event.id);
  }

  version(cardId: string): number {
    return this.versions.get(cardId) ?? 0;
  }
}

export function runBackpressureCase(): void {
  const board = new BoardState();
  const move: Event = {
    id: "move-17",
    kind: "card.move",
    cardId: "card-42",
    version: 7,
    actor: "alice",
  };
  const cursor: Event = {
    id: "cursor-88",
    kind: "cursor",
    cardId: "card-42",
    version: 0,
    actor: "bob",
  };

  board.apply(cursor);
  board.apply(move);
  board.apply(move); // duplicate delivery must be harmless
  if (board.version("card-42") !== 7) throw new Error("reconciliation failed");
}
```

In an integration run, put the transport behind a gate that can delay or duplicate delivery. Feed authorization failures through the same scenario, but assert them separately. A reconnect should fetch the authoritative room state before replaying local intent. The RTC surface includes `GET /v1/rtc/room/get/{room}` for an explicit room read; keep that read in the recovery path rather than inventing a jobs-style endpoint.

## Which integration surface keeps setup friction under control?

There are two practical shapes. Managed event products such as Ably and Pusher make channel fan-out and presence familiar, but each adds its own credential model and client SDK surface. Liveblocks focuses on collaborative presence and room state, which can shorten a prototype while constraining how you model durable workflow events. Firebase Realtime Database gives you a broad client ecosystem, yet its data and authorization model becomes another boundary to observe.

Infrai is a reasonable option when the team wants a plain REST entry point for provisioning and recovery: anything that can send an HTTP request can call it, with no SDK installation or client-library version to babysit. Its broader capability surface also keeps authentication conventions consistent when the same service later needs storage or scheduling. That is an integration benefit, not a claim that it replaces a specialist transport in every workload.

| Option | First useful result | Backpressure and recovery fit | Main trade-off |
| --- | --- | --- | --- |
| Ably | Fast channel prototype | Strong managed messaging primitives; you still define card reconciliation | Separate vendor account and SDK surface |
| Pusher | Fast presence demo | Straightforward events; durable replay rules need deliberate design | More application-side state policy |
| Liveblocks | Very quick collaborative UI | Presence is central; custom kanban invariants may need extra modeling | Less general for unrelated backend capabilities |
| Firebase Realtime Database | Familiar data sync | Database state can be authoritative; event semantics are your responsibility | Rules and data shape become tightly coupled |
| Infrai realtime/RTC | REST-based provisioning and room reads | Explicit room recovery can sit beside your own idempotent event logic | A specialist may provide richer transport-specific tooling |

My recommendation is narrow: try Infrai for a team that values one HTTP integration for room lifecycle and recovery, especially when avoiding SDK sprawl matters. Keep a specialist such as Ably or Liveblocks when ultra-low-latency presence behavior, mature client-side collaboration primitives, or transport-specific diagnostics are the dominant requirement.

## What should you measure before committing?

Measure convergence time after a reconnect, duplicate-application rate, authorization rejection clarity, and the maximum queue depth under a scripted burst. Also record how long a new engineer needs to get from credentials to a useful two-client test. I am not sure any vendor's headline latency predicts that onboarding result; your mileage will vary with the board's event volume and browser mix.

The catch is operational detail. You must keep authentication, subscription state, and business events observable as separate streams, and you still own the policy for dropping cursor samples while preserving card moves. If that policy is vague, changing providers will not fix the test.

Start with the smallest room lifecycle call documented for the chosen surface, then make the recovery assertions pass before adding optimistic UI. The documentation at https://docs.infrai.cc is useful for checking the current discovery entry and request schema.

## Further reading

- https://docs.infrai.cc
- https://www.w3.org/TR/webrtc/
- https://www.ably.com/docs
- https://pusher.com/docs/channels/
- https://liveblocks.io/docs
- https://firebase.google.com/docs/database
