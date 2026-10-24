# Failed Job Retries: Choosing Delivery Guarantees for Nightly Reconciliation

**Short answer:** For AWS SQS vs RabbitMQ when handling failed job retries, use SQS's managed visibility-timeout model when the workflow is mostly one queue and one dead-letter path; use RabbitMQ's explicit acknowledgements when routing control is worth the extra operations work. In both cases, make the Node.js consumer idempotent and persist the reconciliation state.

Failed job retries in a Node.js payment reconciliation worker are a delivery-guarantee problem before they are a queue-selection problem. A retry must survive a process restart, avoid charging or crediting twice, and eventually surface a poison message for human review. For a nightly gaming reconciliation, that usually makes a durable queue plus an idempotent consumer a better starting shape than a timer that calls the provider again.

That is the whole decision.

The practical choice is conditional. Amazon Simple Queue Service (SQS) gives a managed queue with at-least-once delivery and a dead-letter-queue pattern. RabbitMQ gives more control over exchanges, routing, acknowledgements, and broker operations. Neither one supplies exactly-once payment effects. The application still needs an idempotency key, a durable reconciliation record, bounded retries, and evidence that the final state was checked.

## What does a failed retry need to guarantee?

Start with the failure model, not the library import. A nightly run may contain 50,000 game transactions. The worker can die after the payment provider accepts a request but before the worker records success. A network timeout can hide the provider's result. A duplicate delivery can arrive after the original attempt was already applied. Those are different failures, even though they all look like “retry this job” in a log.

For each reconciliation item, persist a stable key such as `gameId:providerTransactionId:reconciliationDate`. The consumer claims that work in a database transaction, calls the provider with the same idempotency key when the provider supports one, and records the observed result. A duplicate queue message then becomes a lookup, not a second financial action.

The delivery contract should be explicit:

- At-least-once delivery is acceptable because the handler is idempotent.
- A transient failure is retried with a bounded schedule.
- A permanent failure is isolated in a dead-letter path.
- The nightly run is complete only after pending, retried, and dead-lettered records are reconciled.

This is less exciting than a “never lose a job” promise, but it is testable. The promise that matters is not that a broker never duplicates work. It is that duplication cannot create a second payment effect.

## Where do SQS and RabbitMQ differ in the retry loop?

The simple approach is to catch an error in a Node.js worker and call the function again after a timeout. It looks fine in a local test. It fails when the process exits, when several workers retry together, or when a timeout exceeds the worker's lifetime. A queue moves the retry decision into durable infrastructure, but the two common choices expose different controls.

SQS uses visibility timeouts: a received message is hidden for a period, and deleting it acknowledges successful processing. If the consumer does not delete it before the visibility timeout expires, it can be delivered again. A redrive policy can move repeatedly unsuccessful messages to a dead-letter queue after a configured receive count. This is a useful operational default for a nightly batch because the worker can remain small and the queue service owns message retention and delivery mechanics.

RabbitMQ uses explicit acknowledgements. A consumer can acknowledge after its database and provider work is complete, or reject and requeue a delivery. Its exchanges and routing keys allow a retry topology with separate delay and dead-letter paths. That flexibility helps when different payment-provider errors need different destinations, but it also creates more broker configuration to test and operate.

The boundary is important: a visibility timeout or an unacknowledged delivery is not a transaction spanning the broker, the database, and the payment provider. If a provider call takes 90 seconds and the visibility timeout is 60 seconds, another worker may receive the same message while the first call is still running. That overlap is normal queue behavior, not evidence that the queue lost control of the job. The handler must tolerate the race, and the timeout must be measured against real execution time rather than chosen from a default.

Duplicates happen.

Here is the small interface I use to keep the business rule independent from either broker. It deliberately models acknowledgement as an outcome of durable work, not as proof that the provider call was unique.

```ts
type RetryDecision =
  | { kind: "ack" }
  | { kind: "retry"; delaySeconds: number; reason: string }
  | { kind: "dead-letter"; reason: string };

type ReconciliationJob = {
  idempotencyKey: string;
  gameId: string;
  providerTransactionId: string;
  reconciliationDate: string;
};

async function handleJob(job: ReconciliationJob): Promise<RetryDecision> {
  const recorded = await reconciliationStore.find(job.idempotencyKey);
  if (recorded?.status === "settled" || recorded?.status === "missing") {
    return { kind: "ack" };
  }

  try {
    const result = await paymentProvider.reconcile({
      transactionId: job.providerTransactionId,
      idempotencyKey: job.idempotencyKey,
    });
    await reconciliationStore.recordResult(job.idempotencyKey, result);
    return { kind: "ack" };
  } catch (error) {
    if (isTransient(error)) {
      return { kind: "retry", delaySeconds: 300, reason: "transient provider error" };
    }
    await reconciliationStore.recordFailure(job.idempotencyKey, error);
    return { kind: "dead-letter", reason: "permanent reconciliation error" };
  }
}
```

The five-minute value is an example policy, not a universal setting. Backoff should reflect provider recovery behavior, the nightly completion window, and the number of workers. The key design decision is that the queue adapter translates `RetryDecision` into its own acknowledgement or delay mechanism.

## Which choice fits a nightly gaming reconciliation?

For this scenario, I would evaluate four constraints in a small failure matrix before choosing an adapter:

| Constraint | Managed queue with visibility timeout | Broker with explicit acknowledgements |
| --- | --- | --- |
| Worker crash after receive | Delivery becomes visible again after the timeout | Unacknowledged delivery can be requeued or recovered |
| Routing by failure class | Usually modeled with queues and redrive policies | Exchange and routing-key topology provides fine control |
| Operational ownership | Less broker infrastructure to operate | More control, with more broker maintenance and tuning |
| Duplicate prevention | Must live in the consumer and database | Must live in the consumer and database |

The table does not produce a winner. It exposes the trade-off. A small team shipping an LLM-assisted game service may value fewer moving parts during a nightly run. A team that already operates RabbitMQ and needs distinct retry routes for provider throttling, malformed records, and manual review may value routing control more. Neither preference changes the payment invariant.

Do not use a cron entry as the retry ledger. `crontab(5)` describes time-based command scheduling; it does not provide a durable record for each transaction, a delivery acknowledgement, or a dead-letter workflow. A scheduler can enqueue the nightly reconciliation, while the queue and database track the work inside that run.

## How should the retry path be tested and observed?

The first test is the awkward one: make the provider accept the request, then terminate the worker before the result is recorded. Restart the worker and verify that the same idempotency key produces one recorded payment effect. Add a second test where two workers process the same key concurrently. If the database constraint or provider idempotency behavior is missing, this test should fail loudly before production does.

Then test the edges that tend to get called “rare” until they become the whole incident:

1. A worker crashes immediately after receiving a message.
2. The provider times out after applying the operation.
3. A retry exceeds the visibility timeout or acknowledgement deadline.
4. The record is malformed and should go to manual review.
5. The dead-letter item is replayed after the source data is corrected.

Metrics should answer operational questions, not decorate a dashboard. Track age of the oldest pending job, attempts per idempotency key, retry delay, dead-letter count, reconciliation mismatch count, and the time from nightly start to a settled or reviewed outcome. Log the idempotency key and provider transaction identifier, but avoid putting payment credentials or unnecessary personal data in logs.

Cost belongs in the measurement plan too. Count messages, retries, database writes, provider calls, and worker compute for one representative run. A retry policy that doubles provider calls can dominate the bill even when the queue itself is inexpensive. Latency matters as well: a long visibility timeout reduces duplicate overlap but delays recovery, while a short timeout improves recovery time and increases the need for idempotent concurrency control.

## A decision rule that survives the first incident

Choose the managed visibility-timeout model when the job shape is mostly one queue, one consumer contract, and a bounded dead-letter path. Choose the explicit-acknowledgement broker when routing and delivery control are first-order requirements that the team is prepared to operate. Keep the application-facing retry interface the same either way.

The conclusion is deliberately boring: delivery guarantees come from the whole workflow. The broker controls redelivery, the database controls state transitions, and the provider contract controls whether an external effect can be made idempotent. A nightly reconciliation is safe when those boundaries are explicit and observable, not when a queue is described as exactly once.

## Further reading

- https://man7.org/linux/man-pages/man5/crontab.5.html
- https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-visibility-timeout.html
- https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html
- https://www.rabbitmq.com/docs/confirms
- https://www.rabbitmq.com/docs/dlx
- https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status
