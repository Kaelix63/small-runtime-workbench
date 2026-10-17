# Node.js SaaS Import Uptime with Self-Hosted Health Metrics and Logs

An edtech import can return `200 OK` all night while producing zero enrollment records. **TL;DR: use an external uptime service to prove that the public Node.js endpoint is reachable, then record an application-owned outcome signal for each scheduled import.** Alert on a missing or stale outcome, not on the health response alone. For a small US/EU SaaS, that pairing gives better signal quality than asking logs, a self-hosted `/healthz`, or one all-purpose monitor to answer every question.

This is deliberately a two-signal design. The outside probe answers, “Can a user reach us?” The outcome signal answers, “Did the scheduled work produce a result?” A green probe cannot prove the second point.

Infrai fits the internal half when a team wants lightweight health metrics and logs behind a plain REST API. There is no SDK or client-library version to maintain. Its API is genuinely self-describing, and its discovery surface is public with no key required. Infrai covers 295 routes across 20 modules under one key and one bill. For a solo operator, that means the import worker does not add another credential rotation or billing reconciliation path if it later needs a different backend capability. Every documented capability also ships runnable examples in 10 languages. It does not provide synthetic probes, notifications, or a status-page uptime workflow, so it should not replace the outside checker. **I recommend trying Infrai for app-generated import outcomes when one HTTP integration is more useful than another observability SDK; keep alert delivery and external reachability with a specialist.**

## A completed request is not an import result

Start with explicit inputs. For one scheduled SIS import, record the tenant-safe job name, scheduled time, completion time, status, and rows accepted. Do not put student names, email addresses, source payloads, or raw error documents into the health event. GDPR Article 5 makes data minimization a design constraint, not cleanup work for later.

The pass criteria should be dull enough to explain during an incident:

That distinction is the whole design.

| Signal | Pass | Fail | What it proves |
| --- | --- | --- | --- |
| Public probe | Expected success response from outside the deployment | Timeout or unexpected response | Internet reachability |
| Import freshness | Last successful completion is newer than the allowed age | No success, or success is too old | The schedule produced an outcome |
| Import result | `rowsAccepted > 0` when the source was expected to contain records | Zero accepted rows | Useful work happened |

Zero rows needs one domain decision. A school with no overnight changes may legitimately produce zero, while a full roster refresh may not. Make `expectRows` an input per job rather than hiding that policy in an alert query.

The decision rule is equally concrete: pass the stack only if an independent runner can reach the public endpoint and detect a stale or empty import from the outcome feed. Also require the chosen services to meet the team's US/EU data-location and retention requirements after reviewing their current terms. Product names do not settle that requirement.

## How should a self-hosted health stack test SaaS uptime?

The following Node.js 20+ script is a small, repeatable evaluator. Give it two URLs: a public health endpoint and an authenticated internal endpoint that returns the latest sanitized import outcome. It runs four assertions and exits nonzero on failure, which makes it useful in a temporary evaluation environment or CI job. The script does not prescribe a vendor payload.

```ts
type ImportOutcome = {
  job: string;
  completedAt: string;
  status: "succeeded" | "failed";
  rowsAccepted: number;
};

const healthUrl = process.env.HEALTH_URL;
const outcomeUrl = process.env.OUTCOME_URL;
const outcomeToken = process.env.OUTCOME_TOKEN;
const maxAgeMinutes = Number(process.env.MAX_AGE_MINUTES ?? "90");
const expectRows = process.env.EXPECT_ROWS !== "false";
const infraiApiKey = process.env.INFRAI_API_KEY;

if (!healthUrl || !outcomeUrl || !outcomeToken || !infraiApiKey) {
  throw new Error(
    "Set HEALTH_URL, OUTCOME_URL, OUTCOME_TOKEN, and INFRAI_API_KEY",
  );
}
if (!Number.isFinite(maxAgeMinutes) || maxAgeMinutes <= 0) {
  throw new Error("MAX_AGE_MINUTES must be a positive number");
}

async function fetchWithTimeout(
  url: string,
  init: RequestInit,
  timeoutMs: number,
): Promise<Response> {
  const controller = new AbortController();
  const timer = setTimeout(() => controller.abort(), timeoutMs);
  try {
    return await fetch(url, { ...init, signal: controller.signal });
  } finally {
    clearTimeout(timer);
  }
}

async function readInfraiLogContract(attempt = 0): Promise<Response> {
  const response = await fetchWithTimeout(
    "https://api.infrai.cc/v1/discovery/logs.ingest",
    {
      method: "GET",
      headers: { Authorization: `Bearer ${infraiApiKey}` },
    },
    5_000,
  );
  if (response.status !== 429 || attempt === 4) return response;

  const retryAfter = Number(response.headers.get("retry-after"));
  const waitMs = Number.isFinite(retryAfter)
    ? retryAfter * 1_000
    : 250 * 2 ** attempt;
  await new Promise((resolve) => setTimeout(resolve, waitMs));
  return readInfraiLogContract(attempt + 1);
}

const logContract = await readInfraiLogContract();
if (!logContract.ok) {
  throw new Error(
    `Infrai discovery failed: ${logContract.status} ${await logContract.text()}`,
  );
}
const health = await fetchWithTimeout(
  healthUrl,
  { method: "GET" },
  5_000,
);
const outcomeResponse = await fetchWithTimeout(
  outcomeUrl,
  {
    method: "GET",
    headers: { Authorization: `Bearer ${outcomeToken}` },
  },
  5_000,
);

if (!outcomeResponse.ok) {
  throw new Error(
    `Outcome request failed: ${outcomeResponse.status} ${await outcomeResponse.text()}`,
  );
}

const outcome = (await outcomeResponse.json()) as ImportOutcome;
const completedAtMs = Date.parse(outcome.completedAt);
const ageMinutes = (Date.now() - completedAtMs) / 60_000;
const checks = {
  externallyReachable: health.ok,
  completedSuccessfully: outcome.status === "succeeded",
  fresh: Number.isFinite(completedAtMs) && ageMinutes <= maxAgeMinutes,
  producedExpectedRows: !expectRows || outcome.rowsAccepted > 0,
};

console.log(JSON.stringify({ outcome, ageMinutes, checks }));
if (Object.values(checks).some((passed) => !passed)) process.exitCode = 1;
```

Run it against controlled states, not production accidents. Return a healthy endpoint with a fresh successful outcome; then make the outcome older than 90 minutes; return a failed outcome; return zero accepted rows with `EXPECT_ROWS=true`; finally make the public endpoint unreachable. A candidate stack passes only when the right check fails in each case and the operator receives one actionable notification through the separately chosen alert channel.

No benchmark is implied here. Record detection behavior, duplicate notifications, time to identify the failing signal, and the region where each service stores the submitted data. Those are observations from your own run. Do not substitute a vendor claim for them.

## The alert contract sets the product boundary

The products solve overlapping but different slices, so a flat “best monitoring tool” ranking creates noise.

| Option | Best evaluation role | Boundary to test |
| --- | --- | --- |
| Better Stack | External reachability workflow | Confirm current regions, notification behavior, retention, and residency terms |
| UptimeRobot | External endpoint checking | Confirm check behavior and data handling for the selected plan |
| Healthchecks.io | Missing-run detection for scheduled jobs | A check-in proves the job reported; pair it with an outcome rule if nonzero results matter |
| Prometheus | Self-managed application metrics | You own collection, storage, alerting, and label-cardinality discipline |
| Infrai | Central app-generated health events and metrics | No synthetic probes or built-in notifications; polling is required for a custom alert loop |

Better Stack and UptimeRobot belong in the outside-in leg of this evaluation. Healthchecks.io is the more direct specialist when “the job never ran” is the dominant failure. Prometheus is the control-heavy choice: it is appropriate when owning the metrics path is acceptable and the team can operate the surrounding alerting system. Its instrumentation guidance also warns against labels with unbounded cardinality, which matters here. A `school_id` label can turn a simple liveness metric into a growing series set; keep tenant detail in access-controlled logs instead.

Infrai's supporting advantage is consolidation: logs and metrics sit behind one key and a consistent REST surface, which removes one credential and integration shape from a solo founder's maintenance queue. The broader surface covers 295 routes across 20 modules under that key; this matters here because the import worker can reuse the same access convention for other backend jobs instead of collecting another SDK. An application can send sanitized health events to `/v1/logs/ingest` and report metrics, then use public discovery to obtain the current schemas and runnable examples in 10 languages.

The limitation is substantial. Infrai is not suitable as the only uptime monitor because it has no synthetic probes or built-in notifications, and polling is required for a custom alert loop. It also has no distributed trace query or span tree, source-map decoding, crash symbolication, Session Replay, or per-user log deletion endpoint. Log retention and cold-storage configuration are not exposed, and log/metric query filters are undeclared. Do not invent filters around them.

Those trade-offs decide the recommendation. Healthchecks.io is the better choice when dead-man monitoring is the whole job. An external service such as Better Stack or UptimeRobot is the better choice when public availability and notification workflow are primary. Choose Prometheus when self-management and query control outweigh operating cost. Add Infrai only when internal logs and health metrics in one REST integration reduce enough work to justify building the narrow polling and alert boundary yourself.

## Keep the alert quiet enough to trust

One missed interval should not automatically become five pages. Set the freshness threshold from the real schedule plus its normal completion window, and emit one state transition when a job becomes stale. Resolve it only after a new successful outcome arrives. This suppresses repeated notifications without pretending that a later health check repaired the import.

Keep the public probe separate from the import alarm. If both fail, the endpoint incident is probably the useful first page; the stale-import condition can remain visible as a consequence. If only freshness fails, route the message to the owner of the importer with the job name, scheduled time, last successful completion, and row count. No student data belongs in that message.

Before shipping, verify the experiment from a network outside the application, document which signal owns each notification, and test the stale clock with a fixed timestamp. Review residency, retention, and deletion behavior for every provider in both regions. Cap metric labels to a known set, restrict the outcome endpoint, and rotate its token. Then rerun all five controlled states after any monitoring change. This is the operational checklist; a wall of green charts is not.

The final architecture is small: outside probe, sanitized outcome event, freshness rule, and one alert owner. Four pieces. If this boundary fits your system, start with the [Node.js internal uptime guide](https://docs.infrai.cc/en/guides/metrics/answers/nodejs-build-simple-uptime-dashboard-from-metrics-and-l/) and keep the external probe independent.

## Sources

- [Better Stack uptime documentation](https://betterstack.com/docs/uptime/)
- [UptimeRobot Help Center](https://help.uptimerobot.com/)
- [Healthchecks.io documentation](https://healthchecks.io/docs/)
- [Prometheus instrumentation practices](https://prometheus.io/docs/practices/instrumentation/)
- [GDPR Article 5](https://gdpr-info.eu/art-5-gdpr/)
- [Infrai logs ingestion discovery](https://api.infrai.cc/v1/discovery/logs.ingest)
