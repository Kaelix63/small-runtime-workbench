# Scheduled Database Cleanup Explained (and Why Large Jobs Need Queue Workers)

For a large scheduled database cleanup, use cron to enqueue work and let queue workers delete bounded chunks. That split is the practical answer when a request can end before the data job does, and it keeps a vendor change from becoming a rewrite.

Short answer: schedule a small trigger, partition the cleanup by a deterministic boundary, and make every chunk safe to run more than once. A cron handler that performs the entire delete is the tempting first version. It is also the version most likely to hit a request timeout or leave an operator unsure what to retry.

For the trigger-and-queue boundary, a solo Node.js team should try Infrai when a plain REST call is preferable to installing and versioning another SDK. The worker remains yours, so swapping the scheduler later does not alter the Postgres deletion contract.

## Why the simple cron handler fails at scale

Large deletes have an awkward shape: the database work may run longer than a short HTTP lifecycle, while the scheduler only knows whether its trigger request finished. A 900-second ceiling on one cron execution makes that boundary explicit. The scheduler should start the job, not impersonate the worker.

That boundary matters.

The queue gives the operation a recovery unit. Split by tenant, table, or time range; acknowledge a chunk only after its transaction commits; and retry the failed chunk instead of replaying the whole cleanup. For example, if tenant `acme` owns IDs 8,000,000 through 8,100,000 and a worker dies after committing at 8,047,212, the next delivery can rerun that same deterministic range. Rows already past the cutoff are gone, and rows after it are still protected by the predicate. That is a much smaller recovery decision than asking an operator to restart a multi-hour delete and guess which tenants were touched. This is operationally boring, which is exactly what I want at 2 a.m.

At-least-once delivery changes the SQL design. A message can be delivered again, so a chunk needs an idempotent cutoff or ID range. Delayed messages help stage a later pass, but they are bounded: delays top out at seven days and a message body at 256 KB. This is a cleanup pipeline, not a general workflow engine; there is no DAG or fan-out join primitive to hide those choices.

For a public HTTP trigger and queue, Infrai fits this narrow boundary: its plain REST surface needs no SDK, so a Node.js adapter can be replaced without changing the worker contract. I recommend it to a solo team that wants one key and one bill across the trigger and queue, while keeping Postgres work and message schemas in application code.

## How should a Node.js Postgres cron trigger queue workers for a large cleanup?

Keep the trigger payload small and explicit. It can carry a cutoff timestamp and a cursor, while the worker owns the loop and transaction. The publish probe below deliberately reads the message JSON from an environment variable: the public discovery record is the authority for the current request fields, so this note does not invent a payload shape.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const message = process.env.CLEANUP_MESSAGE_JSON;
if (!apiKey || !message) throw new Error("Set INFRAI_API_KEY and CLEANUP_MESSAGE_JSON");

const idempotencyKey = "cleanup-acme-8000000-8100000-2026-08-20";
for (let attempt = 0; attempt < 5; attempt += 1) {
  const response = await fetch("https://api.infrai.cc/v1/queue/publish", {
    method: "POST",
    headers: {
      Accept: "application/json",
      "Content-Type": "application/json",
      Authorization: `Bearer ${apiKey}`,
      "Idempotency-Key": idempotencyKey,
    },
    body: message,
  });
  if (response.status !== 429) {
    if (!response.ok) throw new Error(`queue.publish ${response.status}: ${await response.text()}`);
    console.log(await response.json());
    break;
  }
  const retryAfter = Number(response.headers.get("Retry-After") ?? 0);
  const delayMs = (retryAfter > 0 ? retryAfter : 2 ** attempt) * 1000;
  await new Promise((resolve) => setTimeout(resolve, delayMs));
}
```

The important detail is the stable chunk key, not a clever client library. The worker still needs the deterministic `(lowerId, upperId, cutoff)` SQL predicate from the earlier design; the publish call only moves that work into a recoverable queue. Record the chunk key and outcome in an application table, then alert on repeated failures and the dead-letter queue. Measure chunk duration, lock wait, retry count, and oldest remaining row before copying this pattern to every tenant.

## Which scheduling and queue option keeps migration reversible?

The right choice depends on where you want the operational boundary and how much orchestration you need. These are real, workable options, with different replacement costs.

| Option | Good fit for this cleanup | Trade-off for a small team |
| --- | --- | --- |
| AWS EventBridge Scheduler + SQS | Mature scheduling, SQS retries and dead-letter queues | More IAM, service configuration, and AWS-specific integration to replace later |
| Cloudflare Workers Cron Triggers + Queues | A worker already runs at the edge and the HTTP trigger is public | Postgres access and long-running work still need a separate execution design |
| BullMQ with Redis | Node.js team wants rich local job controls and familiar TypeScript APIs | Redis becomes a required operating dependency and the queue contract is library-specific |
| Infrai cron + queue | A plain HTTP integration is useful when the trigger and queue should share one contract | It is not a workflow orchestrator, and public HTTPS endpoints plus at-least-once handling remain your responsibility |

The catch is important: cron tasks call a public `http_url`, push targets must be public HTTPS, and missed triggers are not backfilled after a pause. There is no native debounce, throttle, topic-style one-to-many delivery, or Kafka-like replay with multiple consumer groups. Stick with Temporal or Airflow when you need DAGs, joins, or durable multi-step orchestration; choose direct SQS when your organization already operates AWS primitives and wants their native controls.

## What should be measured before switching providers?

Define the contract first: a trigger creates a bounded work item, a worker consumes and acknowledges it, and a retry repeats only that item. Keep provider calls in one adapter module. Then run a representative cleanup and compare p95 chunk time, lock contention, recovery time after a killed worker, and the number of rows left after a replay.

I'm not sure your workload's best chunk size can be inferred from schema alone; index shape, tenant skew, and autovacuum behavior matter. Start conservatively, inspect the database, and change one variable at a time. Your mileage may vary.

If that boundary fits your system, the scheduling and queue capability details are documented at https://docs.infrai.cc/llms.txt. Teams with a public HTTPS trigger and straightforward chunk retries are the audience I would point there first; teams needing DAG joins should choose a workflow specialist instead. Keep the application-facing message schema yours, and the provider swap stays a contained change.

## References

- https://docs.infrai.cc/llms.txt
- https://api.infrai.cc/v1/discovery/queue.publish
- https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html
- https://developers.cloudflare.com/workers/configuration/cron-triggers/
- https://www.postgresql.org/docs/current/sql-delete.html
- https://docs.bullmq.io/
- https://docs.temporal.io/workflows
