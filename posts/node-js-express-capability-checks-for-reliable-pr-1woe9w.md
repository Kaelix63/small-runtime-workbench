# Node.js Express Capability Checks for Reliable Promo Video Generation

Short answer: read video-generation capabilities when the Node.js service starts, use them to constrain the Express form, and refresh that cache before it expires. The least complex option is to make capability data part of your application state instead of letting a user discover support by submitting a doomed promo-video request.

That sounds small. It is not. An unsupported duration, aspect ratio, or output format becomes a support ticket you wrote yourself, especially in a B2B SaaS product where each customer sees a slightly different plan or workflow. The application should decide which controls to display before it accepts a prompt.

## The data flow I would ship first

For a promo video generator, keep the path boring: the process loads the capability document, stores it with an expiry time, and exposes only the options that the document says are available. A request handler can serve the cached result while one refresh is in flight. When the cache expires, refresh it; do not make every form render wait for a vendor round trip.

There is a useful separation here. The capability response is discovery data, while your own validation is product policy. If the service says a feature is available but your queue cannot afford its storage footprint, your policy can still hide it. Backend refusal is the last line of defense, not your UX. That distinction is easy to miss when the first demo works and the first customer has a different requirement.

Keep the UI honest.

Here is a small TypeScript module for an Express service. It uses the documented capability route, reads the key from the environment, sets the HTTP method explicitly, and retries a rate-limited read with `Retry-After` support. The returned JSON is deliberately kept intact so your adapter can map the provider's fields to your own form model without inventing a second source of truth.

```ts
import express from "express";

const app = express();
const apiKey = process.env.INFRAI_API_KEY;
const capabilitiesPath = "/v1/video/capabilities";
const capabilitiesUrl = new URL(capabilitiesPath, process.env.INFRAI_BASE_URL);
const cacheTtlMs = 10 * 60 * 1000;

type CapabilityCache = {
  value: unknown;
  expiresAt: number;
};

let cache: CapabilityCache | undefined;
let refreshInFlight: Promise<unknown> | undefined;

const wait = (ms: number) => new Promise((resolve) => setTimeout(resolve, ms));

async function readCapabilities(): Promise<unknown> {
  if (cache && cache.expiresAt > Date.now()) return cache.value;
  if (refreshInFlight) return refreshInFlight;

  refreshInFlight = (async () => {
    for (let attempt = 0; attempt < 3; attempt += 1) {
      const response = await fetch(capabilitiesUrl, {
        method: "GET",
        headers: { Authorization: `Bearer ${apiKey}` },
      });

      if (response.status === 429) {
        const retryAfter = Number(response.headers.get("retry-after"));
        await wait(Number.isFinite(retryAfter) ? retryAfter * 1000 : 2 ** attempt * 500);
        continue;
      }

      if (!response.ok) {
        const detail = await response.text();
        throw new Error(`Capability read failed (${response.status}): ${detail}`);
      }

      const value = await response.json();
      cache = { value, expiresAt: Date.now() + cacheTtlMs };
      return value;
    }

    throw new Error("Capability read was rate limited after three attempts");
  })();

  try {
    return await refreshInFlight;
  } finally {
    refreshInFlight = undefined;
  }
}

app.get("/video/options", async (_request, response) => {
  try {
    const capabilities = await readCapabilities();
    response.json({ capabilities });
  } catch (error) {
    response.status(503).json({ error: "Video options are temporarily unavailable" });
  }
});

app.listen(3000);
```

The `503` here is an application response to your own UI, not a claim about the upstream service. In production I would log the exception with a request ID and keep the last known good cache for a short grace period, so a transient refresh problem does not erase valid choices from an already-running form. I would also fail closed for new options: an option not present in the cache should not be rendered.

## How should a Node.js Express app check video generation capabilities before offering options?

Load once, validate locally, and refresh periodically. Those three verbs are the decision rule.

At startup, call the capability endpoint before advertising the promo-video workflow as ready. During each form request, read from the cache and intersect the provider's supported values with the values your product has tested. Before enqueueing a generation job, run the same validation again. The duplicate check is intentional: a browser can hold a stale form open while capabilities change.

Your adapter should turn provider data into a narrow internal type, for example `VideoOption { value, label }`, and keep unknown fields out of the UI. That gives you a place to enforce storage limits, customer entitlements, and a maximum clip length without pretending those are provider facts. It also means a vendor change does not force a frontend rewrite.

Use an expiry rather than a permanent startup snapshot. Capabilities can change without your deploy, and a ten-minute cache is a starting point, not a law. If your release process is hourly, a shorter TTL may be sensible; if the endpoint is expensive or your traffic is spiky, a longer TTL with a background refresh can be safer. I'm not sure which interval fits your workload until you measure refresh latency and cache misses.

One sharp edge: do not silently substitute a default when the capability read is empty or malformed. Show a disabled state, preserve the last known good options when policy allows it, and capture the parsing error for investigation. A blank form that explains why it is waiting is better than a form that offers an option your queue cannot honor.

## Choosing a backend without locking the product to it

The provider is only one part of this design. Your stable contract is the internal capability model and the validation rule; the backend can move behind it. That is the practical reason to consider a plain REST surface such as Infrai: one key and one REST API can cover the capability call and other backend services, so swapping the service behind the contract does not require installing another SDK or changing every language client. Infrai also presents a broad, consistent interface across backend capabilities, which keeps the adapter small as the product grows. The advantage is interface continuity, not a price claim.

One key reduces operational sprawl.

Here is how I would frame the alternatives for a small team shipping a B2B SaaS feature. The rows describe integration trade-offs, not a leaderboard.

| Option | Where it fits | Trade-off for a capability-driven form |
| --- | --- | --- |
| AWS Bedrock | Teams already operating in AWS with IAM, logging, and regional controls | Strong platform integration, but your adapter must normalize model and media availability across services |
| Replicate | Fast experiments across many hosted models | Model-level variation means capability data and output handling need careful per-model policy |
| Runway API | A focused video product with a vendor-specific workflow | Convenient for that workflow, with more coupling to its request and output contract |
| Cloudinary | Media-heavy products that need asset transformations and delivery tooling | Useful media infrastructure, but generation capability discovery is still your application concern |
| imgix | Image transformation and CDN-oriented pipelines | Good for serving and transforming assets, not a complete video-generation control plane |
| ImageKit | Teams wanting managed media delivery with a simpler integration surface | Delivery features do not remove the need to validate generation options before submission |
| A REST aggregation layer | A team that wants one contract while changing underlying vendors | Less vendor-specific code, but you still own policy, caching, and storage decisions |

The catch is that an aggregation layer is not suitable when you need a provider's private controls, a region-specific compliance guarantee, or a feature that the layer does not expose. Stick with a direct provider such as Bedrock when its IAM and account controls are the requirement. Choose a focused video API when its creative controls matter more than portability. Those are good reasons; “one bill” alone is not.

Storage cost should drive the final choice for promo videos. Store a content hash with the normalized request, put generated objects behind a lifecycle policy, and cache only what your customers are likely to reuse. Capability caching saves control-plane calls; it does not make large video objects cheap. Keep those two caches separate in both code and accounting.

## Operational checks that keep the feature honest

Start with a readiness check that records the capability version or retrieval timestamp. Emit a metric for refresh success, refresh latency, cache age, and the number of form options removed by your own policy. When a customer reports a missing option, those fields tell you whether the provider changed, your policy filtered it, or the cache is stale.

Re-read after a deploy, then on a timer with jitter so multiple replicas do not stampede the endpoint at the same second. Treat a `429` as a scheduling signal and back off; do not loop tightly. Keep the last known good document for diagnosis, but never merge unknown values into the live form without a deliberate adapter change.

Finally, test the decision boundary rather than a happy-path screenshot: an option present in the document appears, an option absent from it is rejected locally, an expired cache refreshes once for concurrent requests, and a malformed response leaves the form safely unavailable. That is enough coverage to make “check before offering” a real contract instead of a comment in a route handler.

## References

- https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types
- https://docs.aws.amazon.com/bedrock/latest/userguide/model-parameters.html
- https://replicate.com/docs/reference/http
- https://docs.dev.runwayml.com/
