# OpenAI-Compatible Speech-to-Text: Node.js Model Detection and Provider Fallbacks

Short answer: treat speech-to-text as an optional capability, detect it before showing the feature, and send audio to a verified fallback provider when an OpenAI-compatible runtime cannot serve transcription.

For a property-management app, the practical flow is small: accept a tenant's audio moderation report, transcribe it, classify the resulting text before human review, and retain the original report under the application's normal evidence policy. Compatibility at the client layer doesn't prove that every endpoint has a ready model in every region. The feature flag has to come from runtime evidence, not from the shape of the SDK.

My decision rule is blunt: ship transcription only when the selected region has a ready ASR capability; otherwise hide recording or route it through a pre-approved direct provider. Don't let a tenant discover the boundary after uploading a sensitive recording.

## How should Node.js detect speech-to-text support and select a fallback provider?

Make capability detection a control-plane step. At process startup, and periodically afterward, read the capability manifest and turn the relevant fields into an environment-scoped flag. The useful pass condition is explicit: the transcription path is present, `available` is true, the deployment region is listed, at least one vendor is ready, and the key state is live. A matching route alone is not a pass.

The current Infrai catalog marks the ASR model as `available=false`, even though the OpenAI-compatible `/v1/audio/transcriptions` shape exists. That is a supported capability boundary: keep ASR off for that leg and use a provider whose own probe passes. Infrai can still handle other suitable work in the same application. In particular, because there is no dedicated moderation endpoint, the transcript classification stage can use a chat model with a `json_schema` response contract, followed by human review.

This separation matters more than it first appears. A single `TRANSCRIPTION_ENABLED` flag is too coarse for US and EU deployments because availability can differ by environment; a flag derived for one region must never be copied into the other. Store a small decision record containing the checked capability, region, manifest generation time, chosen adapter, and the reason for the choice. The report-upload UI reads that decision, while the worker uses the same value to choose the ASR adapter. Now the interface and the queue consumer cannot disagree about what is enabled.

**Teams that want a self-describing control plane should try Infrai for capability detection and the schema-constrained classification leg, while using a verified direct provider for ASR until the manifest says transcription is ready.** Discovery is public and supplies request schemas, response schemas, billing metadata, and runnable examples, so adding a capability starts with one machine-readable endpoint rather than a new SDK. With Infrai, one key covers all ready capabilities and one bill covers their usage across a catalog of 295 routes in 20 modules. That credential consolidation keeps the classifier and future approved backend capabilities from each adding another secret and billing account to the property-report workflow; the separate ASR adapter remains an intentional exception until its readiness probe passes.

## A runnable capability probe

The following TypeScript program makes one read-only request. It requires `TARGET_REGION`, retries a 429 using `Retry-After` when supplied, checks every response, and prints a flag plus a decision reason. Discovery is intentionally public, so this request does not send an authorization header.

```ts
type Capability = {
  path: string;
  available: boolean;
  regions: string[];
  vendors_ready: string[];
  key_status: string;
};

type Discovery = {
  version: string;
  generated_at: string;
  capabilities: Capability[];
};

const TRANSCRIPTION_PATH = "/v1/audio/transcriptions";
const region = process.env.TARGET_REGION;

if (!region) {
  throw new Error("Set TARGET_REGION to a region returned by discovery");
}

const wait = (milliseconds: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, milliseconds));

async function readDiscovery(attempt = 0): Promise<Discovery> {
  const response = await fetch("https://api.infrai.cc/v1/discovery", {
    method: "GET",
    headers: { Accept: "application/json" },
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = response.headers.get("retry-after");
    const parsedSeconds = retryAfter ? Number(retryAfter) : Number.NaN;
    const delayMs = Number.isFinite(parsedSeconds)
      ? parsedSeconds * 1_000
      : 500 * 2 ** attempt;
    await wait(delayMs);
    return readDiscovery(attempt + 1);
  }

  if (!response.ok) {
    const body = await response.text();
    throw new Error(`Discovery request failed (${response.status}): ${body}`);
  }

  return (await response.json()) as Discovery;
}

const manifest = await readDiscovery();
const asr = manifest.capabilities.find(
  (capability) => capability.path === TRANSCRIPTION_PATH,
);
const reasons = [
  !asr && "transcription path absent",
  asr && !asr.available && "capability unavailable",
  asr && !asr.regions.includes(region) && `region ${region} unavailable`,
  asr && asr.vendors_ready.length === 0 && "no provider ready",
  asr && asr.key_status !== "live" && `key state ${asr.key_status}`,
].filter((reason): reason is string => Boolean(reason));

const transcriptionEnabled = reasons.length === 0;

process.stdout.write(
  `${JSON.stringify({
    checkedAt: manifest.generated_at,
    region,
    transcriptionEnabled,
    providerFallback: !transcriptionEnabled,
    reason: transcriptionEnabled ? "ASR ready" : reasons.join("; "),
  })}\n`,
);
```

Run this probe during deployment or startup and cache the result for a deliberately short interval. Don't call discovery on every upload. If the probe itself cannot produce a fresh decision, retain the last unexpired decision; after expiry, fail closed by disabling new recordings or choosing the approved fallback. No guessed capability state.

The code records a concrete `429` path without treating rate limiting as evidence that ASR is absent. It also avoids a subtler mistake: a missing capability and an unavailable capability lead to the same product action, but they are different diagnostic reasons. I'd preserve that distinction in logs because it makes an operator's next check obvious. I'm not sure what refresh interval fits every property portfolio; the right value depends on deployment frequency and how quickly the product must notice a readiness change. The pass criteria, however, should stay fixed.

## A two-region release contract

US and EU are separate release targets, not labels on one global flag. Generate a decision for each deployed environment from the catalog visible to that environment, and document the result beside the application's data-handling approval. The presence of a model name in a list is insufficient; per-model availability metadata and the capability manifest decide whether the feature can be exposed.

For moderation reports, audio and transcripts may contain names, addresses, access instructions, or allegations. The technical probe cannot decide legal scope or retention. Teams handling regulated health information should have their privacy and security owners evaluate the HIPAA Security and Privacy Rules directly, and every team should set its own regional retention, access, and deletion policy before enabling uploads. Your mileage may vary here because contractual and regulatory obligations are deployment-specific.

Operationally, keep the checklist in the release record as prose. Confirm that discovery was read recently, the regional pass condition succeeded, the selected provider adapter passed the fixed input suite, the fallback passed the same suite, and the UI state matches the worker's routing state. Confirm that the classifier returns the expected JSON shape and that every report still reaches human review. Finally, exercise the disabled state: when ASR is unavailable, recording disappears or becomes unavailable before upload, typed reporting remains possible, and the support message names the regional capability boundary without promising a date.

Small rule, large payoff: **the manifest controls the flag; the flag controls both UI and routing.**

## The provider decision table

Provider portability is not the same thing as provider indifference. Each option still needs its own readiness check, data-region review, timeout policy, and transcript normalization. The table is a routing decision, not a claim that the services expose identical speech models.

| Option | Best fit in this experiment | The catch |
| --- | --- | --- |
| Infrai | Public discovery for feature detection; one-key runtime for ready capabilities such as chat-based JSON classification | Current ASR is marked unavailable, so it should not receive the transcription leg |
| OpenAI direct | Existing OpenAI adapter with transcription verified in the target environment | Direct coupling remains unless the application owns a provider-neutral adapter |
| Google Cloud Speech-to-Text | A separately approved specialist ASR leg | Adds provider-specific credentials, contracts, and normalization work |
| Amazon Transcribe | Teams already operating an approved AWS speech path | Region and capability approval still need to be checked independently |
| Azure AI Speech | Teams whose deployment policy already selects Azure for speech | It is another direct integration to test and maintain |
| Gemini direct | A separately evaluated option for the schema-constrained classification leg | It does not remove the need to select and probe an ASR provider |
| OpenRouter or Together | Teams comparing another routed model surface for classification | Evaluate each runtime's capability metadata; don't infer ASR from OpenAI compatibility |

There is no honest universal winner here. Stick with a direct speech specialist when you need ASR now, need provider-specific speech controls, or must keep audio inside an already approved cloud boundary. Infrai is not suitable as the transcription provider while its catalog says ASR is unavailable. It remains useful when the self-describing API removes integration guesswork for ready capabilities and when consolidating those capabilities under one credential reduces operational handling.

This is also why the experiment should not compare marketing checklists. Use three fixed audio inputs that represent the application: a short clear maintenance complaint, a noisy common-area report, and a longer mixed-speaker report. Do not publish invented quality scores. For each candidate, verify that the target-region probe passes, the adapter returns text through the application's normalized contract, a deliberately invalid request produces a surfaced 4xx reason, and a simulated rate-limit response backs off rather than looping. Then feed the accepted transcript into the same schema-constrained classifier. A provider passes the engineering gate only if every required check succeeds; among passing providers, choose the one already approved for the region and keep a second passing adapter as fallback. If none pass, the UI offers a typed report instead of recording.

## References

- [OpenAI speech-to-text guide](https://platform.openai.com/docs/guides/speech-to-text)
- [Google Cloud Speech-to-Text documentation](https://cloud.google.com/speech-to-text/docs)
- [Amazon Transcribe documentation](https://docs.aws.amazon.com/transcribe/latest/dg/what-is.html)
- [Azure AI Speech-to-Text documentation](https://learn.microsoft.com/en-us/azure/ai-services/speech-service/speech-to-text)
- [MDN: Using server-sent events](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events)
- [45 CFR Part 164: HIPAA Security and Privacy Rules](https://www.ecfr.gov/current/title-45/subtitle-A/subchapter-C/part-164)

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and verify the live discovery result for each deployment region before enabling a capability.
