# Shipment Fan-Out — 2-Region Daily SaaS Report Email Cron Webhook

**Short answer:** use one small regional cron webhook to open each reporting window, then put every subscriber delivery behind a durable queue and an idempotency ledger; skip a workflow engine until the report becomes a genuinely multi-step process.

For a logistics SaaS sending a daily shipment report in the EU and US, the scheduler should never send the email itself. Its whole job is to name a reporting window and request work. A worker then reads the shipment updates for that window, fans the report out to subscribers, and records each delivery attempt. This boundary matters more than the scheduler brand because cron is at-most a wake-up signal, while email fan-out needs explicit handling for duplicates, retries, partial completion, and regional time.

Keep the trigger boring.

## Implementation: model the shipment window before choosing a clock

Model the run as data, not as a process that happens to be alive. For example, the EU trigger can request `2026-08-18/eu` and the US trigger can request `2026-08-18/us`. The API inserts a run with a uniqueness constraint on that key, discovers the eligible subscribers, and creates one delivery record per subscriber. A queue message carries the delivery ID, not a rendered email or a huge list of recipients. Workers may then retry independently without rebuilding the whole audience.

That gives the system three useful identities: a reporting-window key, a subscriber-delivery key, and an attempt ID. The first prevents two cron invocations from opening two runs. The second prevents a subscriber from receiving the same regional report twice. The third lets logs distinguish a legitimate retry from a duplicate business action. Don't use a random request ID as the idempotency key; it changes on every retry and proves only that two HTTP requests were different.

Time zones need the same discipline. Store window boundaries as UTC instants, but derive them from an explicit IANA zone such as `Europe/Berlin` or `America/New_York`. A fixed `UTC+1` assumption drifts when daylight-saving rules change. The trigger payload should contain the intended logical date and region, while the server calculates and persists the exact start and end instants once. Late workers then use the stored boundaries rather than asking the clock what “yesterday” means.

The guarantee is practical at-least-once processing with idempotent effects. It isn't exactly-once delivery: a queue can redeliver, a worker can lose its lease, and an email provider can accept a request just before the worker loses the response. The ledger can prevent another deliberate send after acceptance is recorded, but it cannot turn an external email API into a distributed transaction. Product copy often blurs that line. The database should not.

## How should a SaaS daily report email cron webhook work across EU and US?

The example below keeps the scheduling interface deliberately small. It uses an in-memory store so the state transitions are visible, but the two uniqueness checks belong in database constraints in production. The worker claims one delivery, sends it with a stable idempotency key when the downstream API supports one, then marks the row delivered. A real queue supplies visibility timeout or lease behavior around `processOne`.

```ts
type Region = "eu" | "us";
type Status = "queued" | "sending" | "delivered" | "retryable";

type Delivery = {
  id: string;
  runKey: string;
  subscriberId: string;
  status: Status;
  attempts: number;
};

const deliveries = new Map<string, Delivery>();

function openRun(logicalDate: string, region: Region, subscriberIds: string[]) {
  const runKey = `${logicalDate}/${region}`;

  for (const subscriberId of subscriberIds) {
    const id = `${runKey}/${subscriberId}`;
    if (!deliveries.has(id)) {
      deliveries.set(id, {
        id,
        runKey,
        subscriberId,
        status: "queued",
        attempts: 0,
      });
    }
  }

  return runKey;
}

async function processOne(
  id: string,
  sendReport: (subscriberId: string, idempotencyKey: string) => Promise<void>,
) {
  const delivery = deliveries.get(id);
  if (!delivery || delivery.status === "delivered") return;

  delivery.status = "sending";
  delivery.attempts += 1;

  try {
    await sendReport(delivery.subscriberId, delivery.id);
    delivery.status = "delivered";
  } catch (error) {
    delivery.status = "retryable";
    throw error;
  }
}

const runKey = openRun("2026-08-18", "eu", ["sub_17", "sub_42"]);
const pending = [...deliveries.values()].filter(
  (delivery) => delivery.runKey === runKey && delivery.status !== "delivered",
);

await Promise.all(
  pending.map((delivery) =>
    processOne(delivery.id, async (subscriberId, idempotencyKey) => {
      console.log("send", { subscriberId, idempotencyKey });
    }),
  ),
);
```

## Test duplicate triggers and expired leases

Take one synthetic run with 2,400 subscribers and stop worker A after the email provider accepts delivery `2026-08-18/eu/sub_42`, but before the worker records `delivered`. At the same moment, invoke the regional trigger again with the identical run key. The second trigger must find the existing run and create zero additional delivery rows. When worker A's lease expires, worker B may claim `sub_42`; it passes the same stable delivery ID to an email API that supports idempotency. If the provider does not support that contract, the system cannot prove that the first call had no effect, so a rare duplicate remains possible. This drill exposes the actual guarantee without needing a production incident: uniqueness protects the fan-out, conditional claims protect concurrent workers, leases recover abandoned work, and downstream idempotency narrows the final uncertainty. A plain in-memory status change provides none of those atomic boundaries, so production code should claim with a conditional update such as `UPDATE ... WHERE status IN ('queued', 'retryable')`, verify that exactly one row changed, and set a lease expiry. Keep the queue visibility timeout longer than usual processing and extend it for legitimately long jobs; AWS documents that an unacknowledged message becomes visible again when its visibility timeout expires.

Retries are normal.

Retries also need a ceiling. Retry timeouts, connection resets, and rate limits with exponential backoff and jitter, but send permanently invalid addresses to a terminal state. After, say, 8 attempts, move the delivery to a dead-letter path and alert on the oldest unfinished reporting window. Eight is an operating-policy example, not a universal constant; your mileage may vary with provider limits and the time at which a daily report stops being useful.

## Cost boundary: count coordination, not cron invocations

The scheduler comparison gets much smaller once it is separated from delivery, and so does the honest cost model: count the trigger, queue operations, database writes, email calls, retained history, and the engineering time needed to recover a window. GitHub Actions can run a repository workflow on a POSIX cron schedule, but scheduled workflows run from the default branch, the shortest interval is five minutes, and GitHub warns that scheduled runs may be delayed during high load. That can fit an internal or low-stakes report where a repository is already the operational home. It is not a good clock for a customer promise at an exact local minute.

Google Cloud Scheduler can target HTTP endpoints, Pub/Sub topics, or App Engine services, while Amazon EventBridge Scheduler can invoke AWS targets and configure retry limits plus a dead-letter queue. Those are managed trigger surfaces, not end-to-end email delivery guarantees. Apache Airflow adds DAG scheduling and task dependencies; Temporal adds durable workflow execution and workflow-level retry semantics. Each broader engine earns its operational weight only when the job actually needs dependencies, long waits, compensation, or inspectable multi-step state.

Here is the decision table I use for the boundary, rather than for a vendor ranking:

| Job shape | Small trigger plus queue | Workflow engine |
| --- | --- | --- |
| One daily query, render, and fan-out | Good fit when each delivery is independent | Usually extra machinery |
| Several dependent extracts with backfills | Possible, but coordination code grows | Stronger fit |
| Approval or wait lasting hours or days | Awkward state management | Stronger fit |
| Exact local send minute promised by contract | Requires a regional scheduler with an explicit SLA | Still requires checking the engine's scheduling guarantees |
| Subscriber-level retry and audit | Must be built into the delivery ledger | Must still be modeled explicitly |

No scheduler removes the last row. Airflow or Temporal may retain richer execution history, but the application still needs a business key for “subscriber 42 received the EU shipment report for August 18.” Conversely, a cheap trigger is only cheap while the team can understand and operate the ledger, queue, and alerting. Hidden engineering time is a cost too.

## Rollout threshold for orchestration engines

Stick with the small cron-webhook design when the report has a single reporting window, subscriber deliveries are independent, retries can finish within a bounded period, and operators can replay one run from stored state. It is especially reasonable for a solo team because the API contract is tiny and the expensive LLM or rendering step can run only after a subscriber has been selected, rather than inside an always-on orchestrator. Batch shared source data once, but meter any subscriber-specific generation separately so one large tenant cannot hide the cost of thousands of personalized outputs.

It is not suitable when the report becomes a graph: ingest from several partners, wait for late files, branch on validation, request approval, compensate an earlier write, and backfill months of history. At that point, choose a workflow engine and pay for the visibility it provides. Keep Airflow in consideration for data-oriented DAGs and scheduled backfills; consider Temporal for application workflows that wait, retry, and resume across service calls. Those are category-level decision cues, not blanket endorsements, and the final choice depends on the team's existing runtime and operational skills.

There is another hard boundary. If a customer contract demands delivery at exactly 09:00 local time, a scheduler whose documentation permits delay is not enough on its own. Measure trigger lag by region, define the acceptable percentile and late threshold, and select a service whose published guarantees and observed behavior satisfy them. I'm not sure a generic “daily” requirement needs that expense; the contract, support history, and a two-week shadow run would resolve it.

## EU data ownership for late windows

Before launch, run both regional schedules in shadow mode and record `scheduled_at`, `opened_at`, first delivery, last delivery, subscriber count, retry count, and terminal failures. Alert on age, not merely on a failed invocation: a green trigger with 600 queued deliveries is still an incomplete report. Dashboards should group by the immutable run key so an operator can answer which window is late without reconstructing events from request logs.

Deploy the webhook and worker before enabling the schedule. Then invoke the same run key twice and confirm that the delivery count does not grow. Kill a worker after it claims a delivery, let the lease expire, and verify that another worker completes it. Test the spring and autumn time changes with stored clock fixtures. Finally, replay one subscriber and one entire window through authenticated operator commands that call the same application functions as normal processing.

Watch the quiet failures. A source query can succeed with zero shipments because a region filter is wrong; an audience query can fall from 2,400 subscribers to 24 without throwing; a report can render with a stale cutoff. Add range checks against recent history, but don't automatically send when those checks fail. Pause the affected window, expose the reason to the operator, and preserve the same run key for the eventual retry.

Silence can be failure.

Ship when duplicate triggers are harmless, abandoned claims recover, late windows page someone, and a replay is routine. The cron expression is the least interesting part of the system — which is exactly where it should end up.

## References

- https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-visibility-timeout.html
- https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows
- https://cloud.google.com/scheduler/docs/overview
- https://docs.aws.amazon.com/scheduler/latest/UserGuide/what-is-scheduler.html
- https://airflow.apache.org/docs/apache-airflow/stable/authoring-and-scheduling/cron.html
- https://docs.temporal.io/workflows
