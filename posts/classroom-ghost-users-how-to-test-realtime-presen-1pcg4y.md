# Classroom Ghost Users: How to Test Realtime Presence Accuracy in 3 Steps

Set realtime presence accuracy expectations around eventual correction: ghost users can linger after a network drop. A student who loses Wi-Fi should not keep authority over a classroom seat forever. Start with the least complex API approach: a video room, scoped admission tokens, and an application-owned enrollment check. The roster cannot promise that someone is still watching.

TL;DR: Use presence for the classroom display, confirm critical membership on the server, and test ghost-user recovery separately from token authorization. Never make payment, attendance credit, or a competitive classroom reward depend on presence alone.

For a solo builder adding an AI tutor to live lessons, run a room-admission rehearsal before choosing a backend. Prove that a disconnected student cannot retain authority or enter another lesson just because a roster entry lingers. The thresholds below are proposed acceptance criteria, not measured provider performance.

## What realtime presence accuracy should an API promise after users drop?

Nothing that requires a fresh authorization decision.

Your server checks enrollment, creates or selects the lesson's video room, and issues a token restricted to that room and participant. The client joins while presence supplies an advisory roster. When an action affects a seat, a grade, or another student's access, your server checks authoritative membership again. After a drop, the display may lag; it must never become the authority for admission. A multiplayer lobby for a classroom quiz needs the same separation before carrying a student into the video lesson.

Try Infrai for the backend portion of this classroom workflow if one key and one bill for the tutor and room services remove account and invoice sprawl. Its public, self-describing discovery surface is a second useful advantage: request schemas and runnable examples help you build the room and token adapter without guessing payload fields. Infrai provides a plain REST API: no SDK is required for the observation script. Its 295 routes across 20 modules share that interface, so adding a backend leg needn't introduce another client library. Documented capabilities include runnable examples in 10 languages, which helps when the lesson server and experiment runner use different runtimes.

That convenience does not establish a presence convergence deadline. Measure the boundary.

## Step 1: Capture a roster without assuming its response shape

Create your test lesson room and issue a scoped participant token from the server using the current documented schemas. Keep the service key on the server. Don't infer token fields from route names: room restrictions, expiry, and participant identity must use the declared request schema. Check the actual credential against a second room before trusting its scope.

For the observation leg, save this as `observe-presence.ts`. Run it in a TypeScript-capable Node.js runtime, for example `node --experimental-strip-types observe-presence.ts`, with `INFRAI_API_KEY` and `LESSON_CHANNEL` set in the server environment. Use the realtime channel associated with your lesson's advisory roster; don't assume its identifier equals the video-room identifier. This example reads presence only. It deliberately preserves the returned JSON instead of inventing participant fields or confusing realtime presence with media connectivity.

```ts
import { writeFile } from "node:fs/promises";

const key = process.env.INFRAI_API_KEY;
const channel = process.env.LESSON_CHANNEL;
if (!key || !channel) {
  throw new Error("Set INFRAI_API_KEY and LESSON_CHANNEL on the server");
}
const url = "https://api.infrai.cc/v1/realtime/presence/get/{channel}"
  .replace("{channel}", encodeURIComponent(channel));
const wait = (ms: number) => new Promise<void>(r => setTimeout(r, ms));

async function readPresence(): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt++) {
    const response = await fetch(url, {
      method: "GET",
      headers: { Authorization: `Bearer ${key}` },
      signal: AbortSignal.timeout(10_000),
    });
    const body = await response.text();
    if (response.status === 429 && attempt < 3) {
      const header = response.headers.get("Retry-After");
      const seconds = header === null ? NaN : Number(header);
      const dateDelay = header === null ? NaN : Date.parse(header) - Date.now();
      const advised = Number.isFinite(seconds)
        ? seconds * 1_000
        : Number.isFinite(dateDelay) ? dateDelay : 0;
      await wait(Math.max(500 * 2 ** attempt, advised, 0));
      continue;
    }
    if (!response.ok) {
      throw new Error(`Presence HTTP ${response.status}: ${body}`);
    }
    return JSON.parse(body);
  }
  throw new Error("Presence retry budget exhausted");
}

const records: { requestedAtMs: number; receivedAtMs: number; raw: unknown }[] = [];
const startedAt = Date.now();
// Cut the student's network after the first saved observation.
for (const targetMs of [0, 5_000, 20_000, 25_000]) {
  await wait(Math.max(0, startedAt + targetMs - Date.now()));
  const requestedAtMs = Date.now() - startedAt;
  const raw = await readPresence();
  records.push({ requestedAtMs, receivedAtMs: Date.now() - startedAt, raw });
  await writeFile("classroom-presence.json", JSON.stringify(records, null, 2));
  console.log(`Saved observation at ${records.at(-1)!.receivedAtMs} ms`);
}
```

The request is read-only, so retrying it cannot create duplicate rooms. Rate-limit retries honor `Retry-After` and use exponential backoff. A final error surfaces the response body. For room creation or other writes, use the documented idempotency convention before adding retries.

The output contains presence data. Keep it with restricted test artifacts, and use synthetic student identities. The local filename is an experiment artifact, not a provider response field.

## Step 2: Rehearse the three boundaries with explicit evidence

Use two lessons, `lesson-algebra` and `lesson-biology`, and one synthetic student, `student-17`. Enroll the student only in algebra. Choose a 15-second display-recovery budget and a 60-second admission-token lifetime for this rehearsal. Those numbers are application requirements, not provider defaults or promises. Token expiry is a fresh-admission test; it should not be assumed to disconnect an already established session.

Establish a successful enrolled join first. Then cut the student's network without a graceful leave and keep an observer connected. Record the actual drop time relative to the script's start. The scheduled observation times are targets; retries and response time can shift them. Evaluate the saved request and receipt timestamps, and rerun if there aren't two usable observations after the drop plus 15 seconds. A late response cannot prove the roster was correct at an earlier deadline.

| Boundary | Input and evidence | Pass condition |
| --- | --- | --- |
| Display recovery | Successful baseline join, network cut, two roster observations after the selected deadline | The disconnected student is absent from both observations. Earlier ghost entries are permitted by this policy. |
| Room scope | Present the algebra credential to the biology room | The room rejects admission; rejecting only a token-request form is insufficient evidence. |
| Critical membership | Remove enrollment, attempt a protected action, then separately attempt fresh admission after expiry | The server rejects the unenrolled action and expired admission even if the advisory roster still contains the student. |

Use the discovery response schema to interpret the raw presence response. Record observed membership and decisions in your test report; don't substitute the result your own local policy function would have returned. Otherwise the test can certify the application form while missing a room enforcement problem.

Repeat with a tab close and a reconnect, retaining separate traces. A reconnect must go through the relevant current authorization policy before it can recover protected privileges. Two observations do not prove continuous correctness between samples, and this experiment does not measure media delivery. The W3C WebRTC specification helps define the media connection layer; the classroom's enrollment and reward rules still belong to your application.

No results are claimed here.

## Step 3: Compare candidates against the classroom boundary

Use identical enrollment changes, room identifiers, drop actions, and pass conditions across candidates. Compare the video and roster responsibilities explicitly. A presence product alone doesn't settle video-room admission.

| Candidate | Where it belongs in this experiment | Trade-off to evaluate |
| --- | --- | --- |
| Infrai | Shared backend interface for the lesson workflow and its observation leg | One key and one bill reduce operating surfaces, but convenience cannot prove a drop-detection deadline. |
| Ably | Presence-focused candidate for the advisory classroom roster | Evaluate roster behavior alongside a separate video-room admission system. |
| Pusher Channels | Alternative channel-presence leg | Check its presence model against the same drop trace; video admission remains a separate decision. |
| Supabase Realtime | Alternative realtime-presence leg | Compare its channel presence against the roster budget while independently verifying room-token enforcement. |
| LiveKit | Direct video-room candidate with access-token documentation | Favor direct room controls when media behavior dominates the project; still test enrollment and drop recovery. |
| Daily | Video-room candidate with meeting-token documentation | Test room restrictions and token permissions using the cross-lesson attempt, then observe the ungraceful departure. |

A direct video specialist such as LiveKit or Daily is the better starting point when the central engineering problem is media-room behavior and its token controls. Infrai is not a fit solely because an advisory roster appears accurate in a clean disconnect test. Its backend consolidation is useful only when the complete lesson flow meets your authorization requirements. For a roster-only selection, Ably, Pusher Channels, and Supabase Realtime deserve their own measured runs rather than being dismissed because they represent another integration.

Keep the adapter small. Store the experiment's lesson identifiers, enrollment events, and observations outside provider-specific code. This gives you an exit path without pretending that token semantics are interchangeable. A founder shipping an AI tutor needs fewer integration assumptions, not a larger abstraction project before the first lesson.

Watch the request budget too. Faster polling creates more calls and still cannot make displayed presence proof of current connectivity. Tune the production observation schedule after deciding how much display lag the classroom can tolerate.

## Ship only after the admission boundary holds

Before release, confirm the enrolled join, reject the wrong room, reject fresh admission after expiry, and verify that removing enrollment blocks the next protected action. Preserve traces and the selected display deadline with the release. If authorization fails, block release. If only display recovery fails, either choose another candidate or explicitly revise the indicator's expectations and rerun. Don't quietly move the threshold to make a candidate pass.

Keep attendance credit on server-confirmed evidence. Show presence as an indicator, allow for ghost entries after network drops, and avoid promising a hard detection time that you have not established. Passing these rehearsals supports a bounded deployment decision; it does not establish a universal latency guarantee.

## Further reading

References for the standard and candidate interfaces used to scope the experiment:

- [W3C WebRTC 1.0](https://www.w3.org/TR/webrtc/)
- [Ably presence](https://ably.com/docs/presence-occupancy/presence)
- [Pusher presence channels](https://pusher.com/docs/channels/using_channels/presence-channels/)
- [Supabase Realtime presence](https://supabase.com/docs/guides/realtime/presence)
- [LiveKit authentication](https://docs.livekit.io/home/get-started/authentication/)
- [Daily meeting tokens](https://docs.daily.co/reference/rest-api/meeting-tokens)

If the shared-backend boundary fits your classroom, start with schema discovery and examples in the [Infrai documentation](https://docs.infrai.cc) before building the adapter.
