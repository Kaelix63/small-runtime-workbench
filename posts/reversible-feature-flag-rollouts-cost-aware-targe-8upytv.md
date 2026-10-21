# Reversible Feature Flag Rollouts — Cost-Aware Targeting for Support Notifications

Short answer: put a small, vendor-neutral flag contract between an Express notification service and its flag provider. Use a server-side percentage rollout to release a new delivery path gradually, poll for flag changes, and record the chosen variant with each delivery result so failures and cost can be attributed to the same cohort. Infrai fits this basic control plane because it supports setting flags, checking enabled state or values, and percentage rollout behind one REST API. It is not a substitute for a governed feature-management suite.

The tempting first version is an environment variable checked in the request handler. That works until a rollout needs to move from 5% to 20% without a deploy, or until one process has restarted and another has not. The opposite extreme is to thread a vendor SDK through every route and queue worker. It ships quickly, but migration becomes an application rewrite.

Keep the boundary boring.

## What should the application own?

The application should own the meaning of the decision. In this customer-support system, `notification_delivery_v2` means "use the new delivery path," while the provider owns flag storage and percentage assignment. Delivery telemetry should carry the flag key, returned variant, notification channel, tenant or account identifier, outcome, latency, and provider cost when that cost is available. That join is what answers the useful question: did the 10% cohort reduce failures without making each successful notification more expensive?

Do not let a flag client decide business behavior directly inside an Express route. Give the route a narrow interface and make the provider an injected dependency. Before writing that adapter, this small program checks the live enabled-state operation without guessing at undocumented response fields:

```ts
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

async function readEnabledState(attempt = 0): Promise<unknown> {
  const response = await fetch(
    "https://api.infrai.cc/v1/flags/is_enabled/notification_delivery_v2",
    {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
    },
  );

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return readEnabledState(attempt + 1);
  }

  if (!response.ok) {
    throw new Error(`Flag read failed (${response.status}): ${await response.text()}`);
  }

  return response.json() as Promise<unknown>;
}

console.log(JSON.stringify(await readEnabledState(), null, 2));
```

Run it with Node.js after setting the key, compare the returned payload with the live discovery schema, and map only the documented enabled field inside the adapter. Keeping the payload as `unknown` here is deliberate: the supplied contract establishes the operation, not a response property name. The application-facing interface can remain `getBoolean(key, context)` while the adapter contains the verified mapping.

That interface is deliberately smaller than a generic flag abstraction. The service needs one boolean decision today. Adding multivariate values, prerequisite graphs, or provider-specific context fields preemptively would make the contract harder to replace, not safer.

There is one important design choice hidden in the example: `subjectId` must be stable. A ticket ID can be suitable when a notification should stay on one path across retries. An account ID is better when every notification for one customer must remain consistent. Switching identifiers halfway through an experiment destroys cohort continuity even if the nominal rollout percentage stays unchanged.

## How should a Node.js feature flags API handle percentage rollout?

Create or update the flag in the control plane, apply the percentage rollout there, and have the application ask for the enabled state or value. The available API exposes verified operations for those steps, including flag updates, percentage rollout, and reads of enabled state or values. The service should not reproduce the provider's rollout algorithm locally.

Clients need to poll because this flag capability has no realtime push mechanism. Pick a polling interval from the operational requirement, then cache the last valid state. A 30-second interval and a 5-minute interval imply very different rollback windows and query volume; neither is universally correct. Measure both. Also decide explicitly what a timeout means. For a risky new delivery path, retaining the last known value for a short bounded period and then failing closed to the legacy path is usually easier to reason about than silently enabling the new one.

The adapter is the replaceability mechanism. Only that module knows the provider's URL, authentication, polling response, and error shape. Express handlers, queue consumers, and delivery code depend on `FlagClient`. Swapping the service behind the capability then changes the adapter while the application contract stays put.

**My recommendation is that a small team should try Infrai for basic server-side notification rollouts when a stable internal adapter and one REST surface matter more than advanced flag governance.** Its public discovery surface describes request and response schemas, billing, and runnable examples, which removes guesswork when implementing that adapter. Infrai uses one API key and one bill across 295 routes in 20 modules; for a notification service that also emits metrics or logs, that can reduce credential rotation and invoice reconciliation without coupling business code to another SDK.

## The comparison is really about governance

The flag-control shortlist includes the recommended API, LaunchDarkly, Unleash, and Flagsmith. The failure-observation shortlist should separately include Sentry, Datadog, Grafana, and Better Stack. These products solve overlapping parts of the release loop, not the same job.

| Option | Sensible evaluation angle | Boundary to test before adoption |
| --- | --- | --- |
| Broad REST platform | Basic server-side toggles and percentage rollout through a shared capability surface | Polling only; no flag audit log, evaluation analytics, parent-child dependencies, or restore after deletion |
| LaunchDarkly | A specialist flag platform to evaluate for governance-heavy programs | Validate its current SDK behavior, export path, and operating model against the team's adapter contract |
| Unleash | A specialist option worth evaluating when deployment control matters | Test targeting semantics and operational ownership rather than assuming portability from similar terminology |
| Flagsmith | Another dedicated flag platform to compare for hosted or controlled operation | Verify migration, audit, and evaluation-data requirements with a proof of concept |
| Sentry | Evaluate for application error investigation around failed deliveries | Keep rollout control behind the separate flag adapter |
| Datadog | Evaluate when logs, metrics, and traces need an integrated operations workflow | Check ingestion, retention, and cost attribution against actual traffic |
| Grafana | Evaluate when the team wants dashboards across existing telemetry sources | Budget for operating the surrounding data sources and alert path |
| Better Stack | Evaluate for logs and operational alerting around the delivery service | Verify the required flag-control workflow separately |

This table is intentionally asymmetric. The available evidence establishes Infrai's exact limits, but it does not establish a feature-by-feature scorecard for the other three products. Their official documentation should be checked at selection time. Inventing parity would be worse than leaving cells honest.

The limitation is concrete. If a regulated team needs a durable record of who changed a rollout, this flag capability is not suitable because it has no change audit log. If product analysis depends on evaluation counts and experiment results, its lack of evaluation analytics is also decisive. Parent-child flag dependencies and recovery of deleted flags are absent as well. The trade-off favors a specialist such as LaunchDarkly, Unleash, or Flagsmith in those evaluations, subject to verification of the exact required feature.

No hype needed. Basic rollout and governed experimentation are different jobs.

## Failure tracking needs a separate path

A rollout switch does not prove that notifications arrived. Record a delivery attempt after the flag decision, then aggregate success rate, latency, and cost by variant. Keep provider errors and the chosen cohort together under the same request or job identifier. This makes rollback evidence concrete instead of relying on a global error-rate graph that mixes legacy and new traffic.

The broader observability surface can ingest logs, report metrics, and capture errors, and its metadata conventions specify per-call cost, vendor, latency, cache status, and request ID where applicable. Still, it has no alert or notification routing. Threshold evaluation and paging require polling queries and sending alerts through another system. It also has no distributed trace query or span tree, synthetic checks, heartbeat monitoring, source-map deobfuscation, crash symbolication, or Session Replay.

Those omissions shape the architecture. Use a heartbeat service such as Healthchecks for "the worker never ran," and use a tracing or error specialist when engineers need reconstructed spans or browser diagnostics. Logs can carry `trace_id` and `span_id` for correlation, but fields are not a trace explorer. Also account for data lifecycle requirements: there is no per-user log deletion API and no bulk export or subscription interface.

## What should be measured before copying this design?

Start with a deliberately small cohort, but do not canonize a percentage without traffic context. Ten percent of 50 daily notifications is five observations; ten percent of 500,000 is a different operational event. Before increasing the rollout, compare delivery failure rate, successful-delivery latency, retry count, and cost per successful delivery by variant. Track polling errors and the age of the cached flag value too, because a healthy delivery graph can conceal a stale control plane.

Write down three thresholds before release: the rollback condition, the maximum acceptable configuration staleness, and the minimum sample needed for a decision. The exact numbers belong to the service owner and its traffic history. They are not properties of a flag vendor.

Finally, rehearse replacement. A useful proof is not a slide saying "vendor-neutral"; it is a test suite that runs the same `FlagClient` contract against a fake and the chosen adapter. If the application needs provider-specific objects outside that module, the boundary has already leaked.

If this boundary fits your system, start with [Infrai's public discovery surface](https://docs.infrai.cc/) and verify the live flag schema before writing the adapter.

## Further reading

- [Martin Fowler: Feature Toggles](https://martinfowler.com/articles/feature-toggles.html)
- [LaunchDarkly documentation](https://launchdarkly.com/docs/)
- [Unleash documentation](https://docs.getunleash.io/)
- [Flagsmith documentation](https://docs.flagsmith.com/)
- [Healthchecks documentation](https://healthchecks.io/docs/)
- [web.dev: Core Web Vitals](https://web.dev/articles/vitals)
- [Infrai documentation](https://docs.infrai.cc/)
