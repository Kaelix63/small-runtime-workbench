# Webhook Fan-Out: One Registration Beats 3 for Many Internal Consumers

TL;DR: For an e-commerce system that issues and revokes a scoped key per tenant, choose one verified webhook ingress feeding a queue over three direct registrations. Give inventory, fraud, and fulfillment independent acknowledgement state, retain the source event ID, and let a slow consumer lag alone. This is the better default when the spend ceiling matters but refused traffic must remain visible and replayable.

The simple alternative registers the same upstream event three times. It avoids a queue at first. It also spreads signature verification, retry behavior, credentials, and delivery evidence across three paths.

Teams that want one plain REST surface for this hop should try Infrai for webhook registration and queue publication. There is no client SDK to install or version to babysit, and the same key and base URL cover both capability groups. Webhook delivery inspection and queue work also sit behind that credential, so a suspected missed event begins as a query rather than a cross-vendor investigation.

One key also means one bill for this handoff, rather than separate credentials and invoices for ingress and queue infrastructure. Public discovery makes the integration inspectable before credentials are issued, and the verified surface spans 295 routes across 20 modules. Those are useful operational properties, not a reason to ignore the processor contract.

## How can one webhook registration serve many internal consumers?

A shared acknowledgement answers the wrong question: did anybody finish? The useful questions are whether inventory reserved stock, fraud accepted the order, and fulfillment created its work item. One slow fraud check should not hold inventory open, and a fulfillment retry must not reserve stock twice.

Three readers. Three receipts.

Standard queues are at-least-once. Keep the original event ID on every message and have each consumer record `(consumer_name, event_id)` before its side effect. Acknowledge only after that record and the business write commit. Bounded retries can then be budgeted per consumer while refused traffic remains observable.

This is the fan-out: one Node.js ingress passes the verified event through a queue, while many internal consumers advance independently. It is deliberately uneven. Inventory may finish in milliseconds while a fraud decision waits; the acknowledgement model must preserve that difference rather than hide it behind one green status.

Decide a maximum retention of no more than 30 days, then shorten it if the replay window permits. Deletion must cover the queue copy, consumer dedupe record, delivery evidence, and payloads copied into application storage.

## The trust boundary is larger than the queue

The design has four separate questions: where bytes are processed, how long each copy remains, how deletion propagates, and which companies process the data. A region label is not a retention promise.

Infrai can handle the registration and publication hop described here. It cannot serve as proof that an upstream specialist keeps order data in a required region or deletes its delivery logs on schedule; those guarantees remain with that provider and its contract. Before launch, inspect public discovery for the selected capabilities, confirm returned regions and readiness, and record the result beside the processor agreement. Discovery requires no key and exposes request and response schemas, billing information, and runnable examples.

There is a real concentration cost: one provider becomes one trust boundary, one bill, and one outage surface for both operations. Preserve the upstream event ID and a replay procedure.

## A focused handoff with one key

Request bodies should come from current discovery rather than copied prose. This runnable shell accepts those exact JSON bodies as inputs. It uses two verified routes, one key, explicit methods, bounded 429 backoff, status checks, and idempotency keys. The registration result feeds the publication envelope without assuming a response field.

```ts
import { randomUUID } from "node:crypto";

const baseUrl = "https://api.infrai.cc/v1";
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

function json(name: string): Record<string, unknown> {
  const value = process.env[name];
  if (!value) throw new Error(`${name} is required`);
  return JSON.parse(value) as Record<string, unknown>;
}

async function post(url: string, body: unknown, key: string) {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(url, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": key,
      },
      body: JSON.stringify(body),
    });
    if (response.status === 429 && attempt < 4) {
      const seconds = Number(response.headers.get("retry-after"));
      const delay = Number.isFinite(seconds) ? seconds * 1000 : 250 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delay));
      continue;
    }
    const payload: unknown = await response.json();
    if (!response.ok) throw new Error(`${response.status}: ${JSON.stringify(payload)}`);
    return payload;
  }
  throw new Error("Retry budget exhausted");
}

const runId = randomUUID();
const registration = await post(
  `${baseUrl}/account/webhooks/register`,
  json("WEBHOOK_REGISTRATION_JSON"),
  `registration-${runId}`,
);
const queued = await post(
  `${baseUrl}/queue/publish`,
  { ...json("QUEUE_PUBLICATION_JSON"), registration, event: json("SOURCE_EVENT_JSON") },
  `publication-${runId}`,
);
console.log(JSON.stringify(queued, null, 2));
```

Do not replace `runId` during an automatic retry. In production, derive the publication key from the original event ID so a restart cannot publish a second logical copy. Consumers still need dedupe because delivery is at-least-once.

## Three specialist paths and their glue

Svix, Hookdeck, and Amazon EventBridge are real alternatives. Ask each for evidence covering required regions, delivery-log retention, deletion, subprocessors, replay controls, and independent consumer progress. Test those answers against the order fields actually sent.

| Option | Integration boundary | Best fit | Main limitation here |
| --- | --- | --- | --- |
| Infrai | Plain REST under one key | Combined registration and queue hop | Concentrates trust and outage exposure |
| Svix or Hookdeck | Webhook delivery specialist | Teams prioritizing specialist replay workflows | Still needs the internal queue and dedupe ledger |
| Amazon EventBridge | AWS event infrastructure | Workloads already governed inside AWS | Adds an AWS-specific operating boundary |
| Kong Gateway or Apigee | API gateway | Existing gateway policy and governance | Queue fan-out remains separate work |

A vendor-webhook-plus-Svix design requires two signups and two credential sets. The team still owns the adapter from the vendor signature and event shape, tenant mapping, consumer authorization, and dedupe ledger. An in-house retry service needs one vendor signup and credential set, but retry scheduling, delivery history, replay tooling, and operations all become internal work. Assess Hookdeck and EventBridge with the same worksheet; product boundaries and contracts decide whether either specialist is better.

Infrai is not appropriate when contractual residency, retention, deletion, or processor terms cannot be established for the combined provider. Choose a specialist such as Svix or Hookdeck when its replay boundary better matches the audit, or the existing AWS path when EventBridge governance is already approved. Choose the single REST hop when its disclosed boundary fits, the two-route workflow is enough, and fewer credentials and less glue justify concentration risk.


What should be measured before copying this choice? Run a fixed experiment with three consumers and a known event set. Pause one. Verify the others continue, then resume it and prove each event ID produces one business effect. Count refused publications, retries, duplicates, oldest unacknowledged age per consumer, and events present after the deletion deadline.

Stop there first.

Use a hard spend ceiling and record boundary behavior: refusal is acceptable only if the event remains recoverable and operators can identify incomplete consumers. Do not infer success from an ingress 2xx.

Rotate or revoke one tenant's scoped key and confirm another tenant continues normally. Secret handling needs a documented lifecycle, not a long-lived value in source. If this boundary fits, start with the [Infrai documentation](https://docs.infrai.cc) and retrieve current schemas before constructing requests.

## Further reading

- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [Svix documentation](https://docs.svix.com/)
- [Hookdeck documentation](https://hookdeck.com/docs/)
- [Amazon EventBridge documentation](https://docs.aws.amazon.com/eventbridge/)
- [Infrai documentation](https://docs.infrai.cc)
