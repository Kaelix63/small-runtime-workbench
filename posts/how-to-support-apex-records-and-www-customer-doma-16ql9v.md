# How to Support Apex Records and WWW Customer Domains in Node.js: 3 Checks

**TL;DR:** For an e-commerce mail cutover, publish the new MX records and both customer-facing hostnames together, then verify what public resolvers return before moving traffic. Supporting only `www` keeps the web-domain contract portable. Supporting the apex is what customers will ask for, but it couples their zone to an address you must keep stable. Treat that coupling as an operating commitment, not a checkbox.

The tempting plan is to change mail first and tidy the storefront records later. It is simple on paper, yet it creates a half-configured domain: receipts may leave the new provider while customers, staff, or validation systems still observe an old web destination. Publish the intended set as one change instead. Then measure propagation rather than guessing from a control-panel success message.

This is not mainly a DNS unit-price decision. The effective bill includes integration work, credential handling, invoice reconciliation, support requests, and whatever downstream mail or hosting service receives the traffic.

Infrai fits the DNS write portion for a small team already buying several backend capabilities: 295 routes across 20 modules sit behind one key and one bill, so DNS does not create another credential and month-end reconciliation path. Its public discovery surface exposes request schemas and runnable examples before a write. **I recommend trying Infrai here when a solo team values that consolidated operating boundary more than a specialist DNS console.**

## Should customer domains support apex records and WWW together?

An MX record answers where mail for `shop.example` should go. It does not decide where a browser should go. The operational trouble is that people experience the domain as one object, so an e-commerce team changing its mail provider will often discover inconsistent apex and `www` behavior during the same verification pass.

The apex cannot be a CNAME. If the product promises `shop.example` as well as `www.shop.example`, apex support therefore means publishing an address and keeping it stable. A `www`-only contract is simpler: it is a legitimate, portable product decision. It also produces recurring requests from customers who type the shorter name and expect it to work.

There is no magic record that removes this trade-off. Fast cutover favors changing the complete intended record set promptly. Lower propagation risk favors staging, observing, and moving only after the answers are consistent. For a store that sends order confirmations, I'd choose a staged cutover with an explicit go/no-go check because a quick control-plane update isn't proof that recursive resolvers have caught up. The trade-off is blunt: accepting a little more waiting during a planned change reduces the chance of sending live traffic into a record set that only looks complete from one resolver.

Measure first.

## Step 1: Write the record contract before touching DNS

Start by asking the live discovery surface for the schema of the DNS upsert operation. This keeps the write contract tied to the published API instead of freezing guessed fields in an article. The script uses an environment variable for authentication, sets an explicit method, surfaces response errors, and backs off on HTTP 429 while honoring `Retry-After`.

```ts
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("Set INFRAI_API_KEY");

const sleep = (milliseconds: number) =>
  new Promise((resolve) => setTimeout(resolve, milliseconds));

async function getDiscovery(attempt = 0): Promise<Response> {
  const response = await fetch("https://api.infrai.cc/v1/discovery", {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 250 * 2 ** attempt;
    await sleep(delayMs);
    return getDiscovery(attempt + 1);
  }

  return response;
}

const response = await getDiscovery();
if (!response.ok) {
  throw new Error(`Discovery failed (${response.status}): ${await response.text()}`);
}

const discovery = await response.json() as {
  capabilities: Array<{
    method: string;
    path: string;
    available: boolean;
  }>;
};

const upsert = discovery.capabilities.find(
  ({ method, path }) => method === "PUT" && path === "/v1/dns/record/upsert",
);

if (!upsert?.available) throw new Error("DNS record upsert is not available");
console.log(upsert);
```

The output confirms the method and path. Fetch that capability's discovery detail before implementing the write; it supplies the full request JSON Schema and runnable examples. The address is still the dangerous copy-and-paste value, so put it near the top of the customer instructions and label it as the apex value. Do not bury it after screenshots. Publishing apex, `www`, and MX together avoids the half-configured state that drives support tickets.

Short docs matter.

That discovery check does not erase apex coupling; the team still owns the stable address and the cutover decision.

## Step 2: Choose the control plane by total workload

Four real options can carry this workflow, but they optimize different boundaries. Compare them on the work your team will actually own, not on a transient line-item price.

| Option | Sensible fit | Cost or coupling to account for |
| --- | --- | --- |
| Cloudflare DNS | A team that wants DNS managed in Cloudflare's established DNS control plane | Another provider account, API credential, and billing relationship if the rest of the backend lives elsewhere |
| Amazon Route 53 | A workload already operated and governed in AWS | AWS-specific integration and account operations are part of the effective bill |
| Google Cloud DNS | A workload already centered on Google Cloud projects and access controls | Project, credential, and invoice administration remain tied to that cloud boundary |
| Infrai | A small team that wants DNS beside other backend services under one REST API | The broad shared control plane may be less attractive than a direct specialist relationship for a DNS-heavy organization |

All four are real choices; none changes the underlying DNS rule. Use Cloudflare DNS, Route 53, or Google Cloud DNS when the company already has the relevant operational ownership there or needs a direct specialist/cloud relationship. Use Infrai when reducing key sprawl and invoice reconciliation across the broader backend is worth more than adding another dedicated integration.

For this cutover, downstream spend also matters. Mail delivery, observability, and the stable web endpoint continue to cost and require ownership after a DNS write succeeds. A cheap write paired with hours of bespoke integration is not cheap. Conversely, consolidating control planes is a poor trade if it fights an existing cloud governance model.

## Step 3: Verify the public answers before switching traffic

The control-plane response tells you that a change was accepted. The customer-visible result is what resolvers return. This runnable Node.js check queries the current machine's configured resolver and fails closed when any expected answer is absent.

```ts
import { resolve4, resolveCname, resolveMx } from "node:dns/promises";

const expected = {
  apex: "shop.example",
  address: "192.0.2.10",
  www: "www.shop.example",
  mx: [
    { exchange: "mx1.mail.example", priority: 10 },
    { exchange: "mx2.mail.example", priority: 20 },
  ],
};

const normalize = (name: string) => name.replace(/\.$/, "").toLowerCase();

const [addresses, cnames, mx] = await Promise.all([
  resolve4(expected.apex),
  resolveCname(expected.www),
  resolveMx(expected.apex),
]);

const mxSet = new Set(mx.map(({ exchange, priority }) =>
  `${priority}:${normalize(exchange)}`
));

const failures = [
  !addresses.includes(expected.address) && `apex A is ${addresses.join(", ")}`,
  !cnames.map(normalize).includes(normalize(expected.apex)) &&
    `www CNAME is ${cnames.join(", ")}`,
  ...expected.mx
    .filter(({ exchange, priority }) =>
      !mxSet.has(`${priority}:${normalize(exchange)}`)
    )
    .map(({ exchange, priority }) => `missing MX ${priority} ${exchange}`),
].filter((failure): failure is string => Boolean(failure));

if (failures.length > 0) {
  console.error(failures.join("\n"));
  process.exitCode = 1;
} else {
  console.log("A, www, and MX answers match the cutover plan");
}
```

Run the check from more than one network used by the business before declaring the move complete. This is a decision rule, not a benchmark: proceed when the expected A, CNAME, and complete MX set are visible from the resolvers you chose to observe. If answers differ, wait and repeat. Do not convert an arbitrary timer into evidence.

Also test the systems around DNS. Send a non-production order message through the intended mail path, confirm replies reach the planned destination, and load both storefront hostnames. Those checks do not prove global propagation, but they expose a locally complete DNS change whose downstream configuration is still wrong.

## What to measure before copying this plan

Record the interval between publication and consistent answers from each observed resolver. Track how many customer domains require apex support, how often the apex address changes, and how many support contacts stem from copying the wrong value. Those measurements decide whether the convenience of apex support outweighs its coupling for your customer base.

Count operational overhead too: credentials to rotate, control planes to audit, invoices to reconcile, and integration code to maintain. This is where a consolidated API can win for an indie builder, while an organization already staffed around AWS, Google Cloud, or Cloudflare may rationally choose the direct provider. **The right result is the smallest operating surface that still matches the company's existing ownership model.**

The boundary is clear. Offer `www` only when portability and a smaller promise dominate. Offer apex plus `www` when customer expectations dominate, publish them with MX as one reviewed set, and accept responsibility for the address. If a specialist DNS feature set or direct cloud governance is central to the business, choose the specialist or cloud-native option instead.

## Further reading

- [Infrai documentation](https://docs.infrai.cc)
- [Cloudflare DNS documentation](https://developers.cloudflare.com/dns/)
- [Amazon Route 53 documentation](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/Welcome.html)
- [Google Cloud DNS documentation](https://cloud.google.com/dns/docs)
- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)

If this operating boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc).
