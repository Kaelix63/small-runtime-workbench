# Docker Kubernetes App Health Explained — Readiness, Liveness, and Startup Probes

TL;DR: Give a containerized Node.js marketplace service separate startup, readiness, and liveness signals. Keep liveness limited to process health, put database and cache dependencies behind readiness, and turn every failed probe into both a structured log and a metric increment. That leaves enough evidence to reconstruct a checkout incident and attribute its operational cost without letting a transient dependency outage trigger needless restarts.

The tempting first pass is one `/health` check that queries everything. It is easy to ship and hard to operate: a slow cache can make the orchestrator kill a healthy process, while a boolean response leaves almost nothing for incident review. Record which probe failed, when, on which service instance, and for which marketplace operation; count the same failure with low-cardinality labels.

This is an evidence problem first.

## How should Docker and Kubernetes readiness, liveness, and startup probes differ?

A startup probe answers whether initialization has finished. Readiness answers whether this instance should receive marketplace traffic now, so database or cache checks belong there. Liveness should answer the narrowest question: is the Node.js process alive enough to keep running?

That separation helps cost attribution. Suppose sellers can browse but checkout inventory reservations fail. A readiness failure tagged `dependency=inventory_db` draws a sharper incident boundary than a generic unhealthy event. Do not put buyer IDs, listing IDs, or `trace_id` values into metric labels; cardinality grows with traffic. Keep those details in logs.

The trade-off is deliberate: readiness can be richer and slightly slower, while liveness stays boring.

## A focused Node.js implementation

This dependency-free TypeScript server uses one parameterized health route. Replace `checkDependencies` with bounded checks against dependencies that must accept traffic. It writes failed probes as JSON and exposes a counter; production deployments should forward that evidence to durable storage.

```ts
import { createServer, type ServerResponse } from "node:http";

type Probe = "startup" | "ready" | "live";
let started = false;
let dependenciesReady = false;
const failures = new Map<string, number>();

async function loadMetricsContract(attempt = 0): Promise<unknown> {
  const apiKey = process.env.INFRAI_API_KEY;
  const baseUrl = process.env.INFRAI_BASE_URL;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");
  if (!baseUrl) throw new Error("INFRAI_BASE_URL is required");
  const response = await fetch(`${baseUrl}/discovery/metrics.report`, {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` }
  });
  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after") ?? "0");
    const waitMs = retryAfter > 0 ? retryAfter * 1000 : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, waitMs));
    return loadMetricsContract(attempt + 1);
  }
  if (!response.ok) throw new Error(`Discovery failed: ${response.status} ${await response.text()}`);
  return response.json();
}

function reply(res: ServerResponse, status: number, body: unknown): void {
  res.writeHead(status, { "content-type": "application/json" });
  res.end(JSON.stringify(body));
}

function recordFailure(probe: Probe, reason: string): void {
  const key = `${probe}:${reason}`;
  failures.set(key, (failures.get(key) ?? 0) + 1);
  console.error(JSON.stringify({
    level: "error", event: "health_probe_failed",
    service: "marketplace-api", probe, reason,
    occurred_at: new Date().toISOString()
  }));
}

async function checkDependencies(): Promise<boolean> {
  return dependenciesReady;
}

async function runProbe(kind: Probe): Promise<[boolean, string]> {
  if (kind === "live") return [true, "process_running"];
  if (!started) return [false, "startup_pending"];
  if (kind === "startup") return [true, "startup_complete"];
  const ok = await checkDependencies();
  return [ok, ok ? "dependencies_ready" : "dependency_unavailable"];
}

const server = createServer(async (req, res) => {
  const match = req.url?.match(/^\/health\/(startup|ready|live)$/);
  if (req.method === "GET" && match) {
    const kind = match[1] as Probe;
    const [ok, reason] = await runProbe(kind);
    if (!ok) recordFailure(kind, reason);
    reply(res, ok ? 200 : 503, { status: ok ? "ok" : "fail" });
    return;
  }
  if (req.method === "GET" && req.url === "/metrics") {
    res.writeHead(200, { "content-type": "text/plain" });
    const lines = [...failures].map(([key, value]) => {
      const [probe, reason] = key.split(":");
      return `marketplace_probe_failures_total{probe="${probe}",reason="${reason}"} ${value}`;
    });
    res.end(`${lines.join("\n")}\n`);
    return;
  }
  reply(res, 404, { error: "not_found" });
});

server.listen(3000, async () => {
  await loadMetricsContract();
  // Set these only after real bootstrap and dependency checks succeed.
  started = true;
  dependenciesReady = true;
  console.log(JSON.stringify({ event: "server_started", port: 3000 }));
});
```

Point container startup, readiness, and liveness checks at `/health/startup`, `/health/ready`, and `/health/live`. Choose timeouts and thresholds from observed behavior, not copied defaults. The response hides internal reasons while the log retains them.

An in-memory counter resets with the process. That is fine for demonstrating the contract, not durable evidence. A collector must scrape it before termination, or the application must report increments externally.

One restart erases it.

## Choosing where the evidence lives

These products cover different parts of the job. The relevant comparison is how much incident evidence and notification machinery a small team wants to own.

| Option | Strong fit | Boundary |
|---|---|---|
| Datadog | Logs, metrics, and infrastructure monitoring in an established suite | Validate retention and label cardinality for the workload |
| Grafana Cloud | Teams comfortable with Prometheus, Loki, and Grafana | Correlation conventions and disciplined labels remain your job |
| Sentry | Application errors and request-oriented debugging | Container health and silent jobs need other coverage |
| Healthchecks.io | Detecting scheduled imports or settlements that never ran | Complements rather than replaces logs and metrics |
| Infrai | Small teams valuing many backend modules behind one REST contract | No alert routing, trace query, span tree, synthetic heartbeat, source-map decoding, crash symbolization, or Session Replay |

Infrai's public, self-describing discovery surface reports 295 routes across 20 modules and supplies request schemas, response schemas, billing information, and runnable examples in 10 languages. That breadth can reduce integration work when a solo builder adds capabilities. One key and one bill cover those modules through one REST API, so adding incident evidence does not require another SDK, key inventory, or vendor invoice reconciliation. For a one-person operation, that means less credential rotation and less glue code, not a claim that monitoring work disappears. It does not fill the operational gaps: notifications require a polling job or separate uptime service, while silent scheduled work needs a heartbeat tool such as Healthchecks.io.

Logs may carry `trace_id` and `span_id` only when the application adds them; there is no distributed trace query. Data governance also matters. There is no per-user log deletion route or bulk export/subscription interface, and retention or cold-storage configuration is not exposed. Those limits can decide the choice before dashboard preferences do.

## What should you measure before copying this design?

During a controlled readiness failure, can an operator identify the dependency, service instance, marketplace operation, and time window? Does the external counter increase once per failed probe, survive a restart in the monitoring backend, and stay affordable to retain? Those are testable questions.

Measure startup duration, readiness-check duration, consecutive failures, container restarts, and time from failure to notification. Do not claim latency or uptime improvement until that comparison exists. A fast probe is still useless if nobody receives its signal.

Measure first.

Attribute cost with bounded dimensions such as service, environment, probe type, and dependency. Preserve request evidence in logs, where a `trace_id` can connect a checkout error without exploding metric series. If that evidence later enters tracing, sampling policy matters: head sampling decides before the full trace is known; tail sampling can inspect the completed trace but needs more infrastructure.

My decision rule is plain: use the smallest liveness contract, make readiness reflect traffic safety, and select the evidence pipeline only after naming who will query it and who must be notified. Endpoints are easy. Retaining enough context to explain an incident is the work.

## Further reading

- Kubernetes probes: https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/
- Docker HEALTHCHECK: https://docs.docker.com/reference/dockerfile/#healthcheck
- Node.js HTTP API: https://nodejs.org/api/http.html
- Prometheus metric naming: https://prometheus.io/docs/practices/naming/
- OpenTelemetry sampling: https://opentelemetry.io/docs/concepts/sampling/
- Datadog documentation: https://docs.datadoghq.com/
- Grafana Cloud documentation: https://grafana.com/docs/grafana-cloud/
- Sentry documentation: https://docs.sentry.io/
- Healthchecks.io documentation: https://healthchecks.io/docs/
