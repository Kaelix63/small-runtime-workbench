# Node.js Video Call API for SaaS Scoped Rooms (Trust Boundary)

**TL;DR:** For a logistics SaaS that combines a dispatch document, collaborative cursors, and a video call, choose the media system by its authorization boundary: the Node.js backend must mint a short-lived grant for one tenant, one review room, one user, and a small set of actions. The browser should receive that grant only after the application has checked membership. It should never receive the signing secret or gain the ability to choose a different room. WebRTC defines browser media behavior, but the application still owns identity, room admission, and cursor authorization.

This decision matters more than a long feature matrix. A polished SDK cannot repair a token that means "join any room," and a private room name is not access control. For a solo builder, the useful test is blunt: can I express the policy once on the server, inspect the resulting claims, and replace the media implementation without rewriting the dispatch domain?

## What should a Node.js SaaS video call API scope?

Very little.

It may ask to join dispatch review `review_8f2`, publish microphone and camera tracks, subscribe to permitted tracks, and send cursor updates for document `loadplan_417`. It must not decide that `review_8f2` belongs to tenant `carrier_north`, grant itself moderation, or turn a read-only guest into a publisher.

Those distinctions create three separate planes. The application plane authenticates the user and decides membership. The media plane carries audio and video through WebRTC. The collaboration plane distributes ephemeral cursor positions and applies document permissions. They may share a visible room, but they should not share an undifferentiated bearer token. A leaked cursor credential should not authorize a camera publication; a media grant should not authorize edits to a load plan.

The plain-language flow is short. The signed-in browser asks the application server for access to a specific dispatch review. The server loads tenant membership and review policy, issues narrow grants, and returns only the browser-safe connection material. The browser then starts media and cursor sessions independently. On reconnect, it requests fresh authorization instead of treating an old socket as proof of membership.

## Mint the narrow grant before wiring the UI

I start with a small domain policy and an adapter boundary. This TypeScript example is runnable with a project-specific `signToken` implementation; the important part is that untrusted request data never supplies the tenant, role, or final permissions.

```ts
type ReviewRole = "dispatcher" | "driver" | "observer";

type SessionUser = {
  id: string;
  tenantId: string;
};

type ReviewMembership = {
  reviewId: string;
  tenantId: string;
  userId: string;
  role: ReviewRole;
  documentId: string;
};

type MediaGrant = {
  subject: string;
  tenantId: string;
  roomId: string;
  canPublish: boolean;
  canSubscribe: boolean;
  expiresAt: number;
};

type CursorGrant = {
  subject: string;
  tenantId: string;
  documentId: string;
  canSendCursor: boolean;
  expiresAt: number;
};

interface MembershipStore {
  find(reviewId: string, userId: string): Promise<ReviewMembership | null>;
}

interface TokenSigner {
  signMedia(grant: MediaGrant): Promise<string>;
  signCursor(grant: CursorGrant): Promise<string>;
}

type JoinBundle = {
  mediaToken: string;
  cursorToken: string;
  expiresAt: number;
};

export async function authorizeReviewJoin(
  session: SessionUser,
  requestedReviewId: string,
  memberships: MembershipStore,
  signer: TokenSigner,
  nowMs = Date.now(),
): Promise<JoinBundle> {
  const membership = await memberships.find(requestedReviewId, session.id);

  if (!membership || membership.tenantId !== session.tenantId) {
    throw new Error("review membership required");
  }

  const expiresAt = nowMs + 5 * 60 * 1000;
  const canPublish = membership.role !== "observer";
  const subject = `${membership.tenantId}:${session.id}`;

  const [mediaToken, cursorToken] = await Promise.all([
    signer.signMedia({
      subject,
      tenantId: membership.tenantId,
      roomId: membership.reviewId,
      canPublish,
      canSubscribe: true,
      expiresAt,
    }),
    signer.signCursor({
      subject,
      tenantId: membership.tenantId,
      documentId: membership.documentId,
      canSendCursor: membership.role !== "observer",
      expiresAt,
    }),
  ]);

  return { mediaToken, cursorToken, expiresAt };
}
```

Five minutes is an example policy choice, not a universal optimum. The right lifetime is shorter than the period in which membership could materially change, yet long enough to tolerate a normal page load and connection attempt. Expiry alone is also insufficient. For immediate removal, the application needs a server-side way to revoke or disconnect an active participant; the exact mechanism belongs behind the adapter.

Notice what is absent from the function: a vendor room object, browser-provided role, and signing key in frontend code. The returned tokens can use different claim formats. Keeping those formats inside `TokenSigner` prevents a future media change from leaking through every route and React component.

## Scope is a security property, not a naming convention

Room identifiers should be opaque and server selected, but opacity only reduces accidental discovery. Authorization must bind the authenticated subject to the tenant and exact room. If the media service has no native tenant claim, the application can encode tenant ownership in its own room registry and reject mismatches before signing. It should still avoid treating a prefix such as `carrier_north_` as the sole check.

The dangerous implementation is often convenient: accept `roomId`, `role`, and `canPublish` from JSON, copy them into a signed token, and call that secure because the signature is valid. The signature then proves only that the backend repeated attacker-controlled input. Consider the full path: an authenticated observer opens developer tools, changes `review_8f2` to a review copied from another tab, changes `canPublish` to `true`, and submits the otherwise ordinary request. If the server signs those fields before loading membership, every cryptographic check can pass while the authorization decision is wrong. The database lookup in the example is the actual boundary because it derives tenant, document, role, and allowed actions from trusted state, then treats the request as no more than a resource locator.

**A grant should describe one identity, one resource, and the minimum actions needed.** For dispatch reviews, publishing and subscribing are different actions. Recording, screen sharing, data messages, room administration, and participant removal are separate again if the chosen system exposes them. Do not issue those capabilities merely because the SDK supports them.

Cursor traffic deserves the same skepticism. Validate document membership at connection time, rate-limit updates, cap payload size, and reject coordinates tied to a stale document revision when the protocol requires revision consistency. Cursor updates are ephemeral UI state; the durable load plan remains governed by the document service. Mixing both into one generic realtime channel makes audit and revocation harder.

## Evaluate the boundary with adversarial tests

An API comparison should begin with tests, not screenshots. Build a thin proof using two tenants, two rooms per tenant, and three roles. Then try to cross every boundary. A user from tenant A must fail to join tenant B's review even after changing a request body. An observer must fail to publish. An expired grant must fail after reconnect. A removed dispatcher must lose active access within the revocation objective the application has chosen.

Keep the matrix small enough to rerun during upgrades:

| Test | Expected decision | Evidence to retain |
| --- | --- | --- |
| Change only the requested room ID | Deny before token signing | Application authorization log |
| Replay an expired grant | Deny at connection | Expiry reason without token contents |
| Ask for publisher access as observer | Issue subscribe-only scope | Decoded non-secret claims in test output |
| Reuse a media grant on cursor transport | Deny due to audience or signer separation | Protocol authentication result |
| Remove an active member | Revoke or disconnect within policy | Timestamped membership and session events |

This exercise reveals the meaningful selection criteria. Can the system restrict join, publish, subscribe, and administration independently? Can the server disconnect a participant after membership changes? Are token claims documented and inspectable without exposing secrets? Can room and participant events be correlated with the application's tenant and review IDs? Does the browser SDK expose reconnect state so the UI can stop presenting stale cursors as live people?

A search for a LiveKit or Daily alternative is a reasonable way to collect candidates, but the names do not answer those questions. Apply the same scoped-room tests to every candidate. This note does not rank those products because the evidence needed for a choice is the result of the application's own policy tests and workload proof, not a generic label.

WebRTC itself does not answer those application questions. Its specification defines browser-facing peer connection, media, and data-channel behavior. Signaling is outside the specification, which is one reason authentication and room semantics vary between implementations. Treat each provider or self-hosted stack as an adapter to test against the same contract, not as the owner of the application's identity model.

Cost still belongs in the proof, just not as the headline. Measure billed media minutes, egress, recording, and idle-room behavior using the dispatch workload you expect. Also count engineering costs: secret rotation, webhook verification, regional deployment, failure investigation, and upgrade testing. A nominal rate is weak evidence when the architecture forces broad grants or leaves revocation opaque.

There is a real limitation to the two-token design: separate media and cursor sessions add issuance, refresh, revocation, and observability work. For a prototype with no sensitive tenant boundary, that overhead may outweigh its benefit. A single transport can be acceptable when its grant still separates resources and actions, while regulated dispatch records or external guests justify the clearer isolation. I choose separation here because a logistics SaaS has two different assets to protect and because browser compromise should have a smaller blast radius. That is a trade-off, not a universal rule.

## Operate the policy you selected

Before launch, log authorization decisions with subject, tenant, review, role, requested action, decision, and policy version. Never log bearer tokens. Track token issuance failures separately from media connection failures, and separate both from cursor transport failures; one blended "realtime error" counter wastes the first minutes of an incident.

Deployment should preserve the adapter contract. Pin SDK versions, rotate signing credentials without placing them in the browser bundle, verify inbound event signatures when events influence application state, and rehearse revocation. Test clock skew around expiry. Exercise browser permission denial, tab suspension, network changes, duplicate joins, and reconnect after membership removal. Cursor state should disappear or become visibly stale when its session dies, while the durable document can recover from its own persisted version.

The final review is prose, not ceremony: confirm that every browser credential has a narrow audience and expiry; confirm that the backend derives tenant, role, and room membership from trusted state; confirm that media and cursor permissions can be revoked independently; confirm that logs identify a decision without capturing secrets; and confirm that the application can swap the media adapter without changing its authorization rule. If any answer is vague, the integration is not ready.

Ship the boundary first.

The call UI can change later.

## Sources

- W3C, *WebRTC 1.0: Real-Time Communication Between Browsers*: https://www.w3.org/TR/webrtc/
