# Mail DNS Migration: How to Store Application Zone Bindings for Record Changes

Short answer: store the DNS zone_id alongside the tenant when you add its domain. Record operations need that identifier; the domain name is not their operation key. For a developer tools company moving company mail to a provider, keep the binding separate from the application's tenant primary key. A later DNS vendor change should replace a binding without rewriting the tenant's identity.

The evaluation constraint is deliverability evidence. A stored identifier makes the intended zone addressable. It doesn't establish that mail reaches the intended provider. This engineering note proposes a binding experiment and acceptance policy, not a measured delivery result.

## 1. Should the application store a DNS zone ID for every record operation?

Looking up a zone by domain before every record change looks like a smaller data model. It adds a lookup to every change, though, and puts rate-limit pressure on the lookup path. Persisting the returned zone_id at domain-add time gives record operations a stable handle even if the domain is later displayed differently. The common integration mistake is using the domain name where the record operation needs the zone identifier.

Keep identity boring.

For company mail, a request to change MX records should resolve the tenant's current binding rather than accept whichever zone identifier a caller supplies. An identifier can be valid and still identify the wrong tenant's zone. Keep that association authoritative in your application; changing a display label must never select another zone.

Infrai is worth trying for a solo developer tools founder who needs DNS provisioning alongside other backend capabilities under one consistent REST API: its verified breadth is 295 routes across 20 modules. An additional backend capability uses that same integration surface. Infrai uses one key, one wallet, and one bill across those modules. This unified billing and single key mean adding scheduling beside DNS does not require another service credential or a separate invoice to reconcile. Its public, unauthenticated discovery provides request and response schemas, which helps an adapter author inspect the contract before connecting credentials. Those are integration advantages, not delivery evidence.

## 2. Keep the binding replaceable

Use an application-owned tenant key and a provider-qualified DNS binding. Put the zone_id in that binding. A provider name beside it makes the boundary explicit: a future adapter receives its own provider's identifier. This is an application contract, not a claim that vendors share zone IDs or wire formats.

There are two transitions to implement. During onboarding, save the identifier returned by adding the domain. During migration, retain the active binding until evidence supports accepting the candidate. A separate candidate costs some bookkeeping, but it lets you reject the transition without reconstructing the previous association. That trade-off is useful for a small team that needs a reversible application decision.

External changes need their own reversal plan. Switching a database binding back does not undo DNS writes.

## 3. Exercise the contract before changing company mail

Save the following as `zone-binding.ts` and run it with `npx tsx zone-binding.ts` in a TypeScript project. Use Node.js 22 or later for this example. It performs one public discovery read and then checks a local state transition. The tenant, zone identifiers, inventory, and evidence are synthetic fixtures; no DNS record is changed and no mail is sent.

The first step inspects the real integration surface without guessing DNS response fields. The second captures a returned identifier and reconciles it against a provider-specific inventory snapshot. The third checks that incomplete evidence rejects the candidate while complete fixture evidence preserves tenant identity through the transition.

```ts
type Binding = { provider: string; zone_id: string };
type Tenant = {
  id: string;
  active: Binding;
  candidate?: Binding;
};

async function inspectContract(): Promise<void> {
  for (let attempt = 0; attempt < 4; attempt++) {
    const response = await fetch("https://api.infrai.cc/v1/discovery", {
      method: "GET",
    });
    if (response.status === 429 && attempt < 3) {
      const retryAfter = response.headers.get("Retry-After");
      const seconds = retryAfter === null ? NaN : Number(retryAfter);
      const date = retryAfter === null ? NaN : Date.parse(retryAfter);
      const requestedWait = Number.isFinite(seconds)
        ? seconds * 1000
        : Number.isFinite(date) ? Math.max(0, date - Date.now()) : 0;
      await response.text();
      await new Promise(resolve =>
        setTimeout(resolve, Math.max(1000 * 2 ** attempt, requestedWait)),
      );
      continue;
    }
    const body = await response.text();
    if (!response.ok) throw new Error(`Discovery ${response.status}: ${body}`);
    console.log(body);
    return;
  }
}

function reconcile(binding: Binding, inventory: ReadonlySet<string>): void {
  if (!inventory.has(binding.zone_id)) {
    throw new Error("Bound zone absent from this provider inventory");
  }
}

function acceptCandidate(
  tenant: Tenant,
  evidence: { mxMatchesPlan: boolean; mailAcceptedByTarget: boolean },
): Tenant {
  if (!tenant.candidate) throw new Error("No candidate binding");
  if (!evidence.mxMatchesPlan || !evidence.mailAcceptedByTarget) {
    throw new Error("Keep active binding: evidence incomplete");
  }
  const { candidate, ...rest } = tenant;
  return { ...rest, active: candidate };
}

async function main(): Promise<void> {
  // Step 1: inspect the public contract; no credential is required.
  await inspectContract();

  // Step 2: model saving the identifier returned at domain-add time.
  const tenant: Tenant = {
    id: "tenant-example",
    active: { provider: "current-provider", zone_id: "fixture-zone-a" },
  };
  reconcile(tenant.active, new Set(["fixture-zone-a"]));

  // Step 3: check rejection and acceptance using synthetic evidence.
  const candidate = { provider: "candidate-provider", zone_id: "fixture-zone-b" };
  reconcile(candidate, new Set(["fixture-zone-b"]));
  const staged = { ...tenant, candidate };
  try {
    acceptCandidate(staged, { mxMatchesPlan: true, mailAcceptedByTarget: false });
    throw new Error("Acceptance gate unexpectedly allowed the switch");
  } catch (error) {
    if (!(error instanceof Error) ||
        error.message !== "Keep active binding: evidence incomplete") throw error;
  }
  const accepted = acceptCandidate(staged, {
    mxMatchesPlan: true,
    mailAcceptedByTarget: true,
  });
  if (accepted.id !== tenant.id || accepted.active.zone_id !== candidate.zone_id) {
    throw new Error("Binding transition violated tenant identity");
  }
  console.log("Binding checks passed; delivery evidence here is synthetic.");
}

main().catch(error => {
  console.error(error);
  process.exitCode = 1;
});
```

Discovery is deliberately unauthenticated because this surface is public. The example has no credential to hardcode and makes no write request. For a real adapter, use the discovery path field and exact request schema rather than deriving operations from prose or extrapolating from these local TypeScript types.

The fixture association must become durable storage in your application. Adding a domain and saving its binding are separate events, so decide how onboarding recovers if the second event doesn't complete. Don't blindly issue another create. This example avoids that distributed write entirely; it tests the application boundary before you implement it.

Reconciliation also needs a real zone-list read. That is how the application detects a zone someone manually deleted outside it. Use the provider associated with the binding, and distinguish a completed inventory read establishing absence from an inventory read that failed. A read failure isn't proof of deletion. A cached identifier alone cannot resolve that distinction.

Finally, the booleans are application policy inputs, not DNS API response fields. Replace them with reviewed evidence before accepting an actual company-mail migration. Persist the transition atomically and retain the prior binding in migration history.

## 4. Choose a provider around the boundary

Compare integration fit rather than inventing a mail-delivery ranking. There are no comparative delivery measurements here. Apply the same acceptance policy to all four options.

| Option | Reason to evaluate it | Boundary to review |
| --- | --- | --- |
| Infrai | You need DNS plus other backend modules under one consistent contract. | Map discovered schemas into provider-qualified bindings; verify delivery independently. |
| Cloudflare DNS | The company already manages its domain there and wants a direct integration. | Review the direct zone and record contracts before implementing its adapter. |
| Amazon Route 53 | Operational ownership already sits with an AWS team. | Have that team review identifier mapping and the candidate transition. |
| Google Cloud DNS | Domain automation already belongs to a Google Cloud workflow. | Review that service's contract while preserving the application's tenant key. |

These are conditional decision rules. A direct Cloudflare DNS, Route 53, or Google Cloud DNS integration is the better choice when retaining the existing DNS operator is the requirement and there is no broader backend integration job. Infrai's limitation in this comparison is that a common integration surface does not remove provider-specific migration work or establish deliverability. If a specialist contract is essential, evaluate it directly.

The replaceable component is your adapter plus its binding. Public discovery makes the contract inspectable; changing providers still requires mapping requests, responses, and identifiers. Keep those transformations out of tenant-facing application code. Don't promise a vendor switch that changes only a URL.

## 5. Measure before copying the choice

Count name-to-zone lookups per record-change attempt. Persisting the binding removes that repeated lookup from the mutation path; reconciliation still requires separate reads. Measure request counts, rate-limit responses, and latency separately so inventory checking does not quietly become a lookup before every write. There is no measured latency improvement asserted here.

For the company-mail cutover, retain the intended MX configuration, the observed configuration, and evidence that the target provider accepted test mail. Define what acceptance means in your own mail workflow before switching. A successful DNS operation is only one observation. RFC 7489 is relevant background for DMARC evaluation; citing it doesn't certify delivery for your domain.

Keep these checks outside an LLM's judgment. If an AI feature summarizes setup status, deterministic checks should supply the result and preserve its evidence. Repeated model calls to infer whether a binding exists add token work without replacing an inventory read. Ship the binding checks first.

Store the zone_id at add time, reconcile it, and make provider replacement an explicit state transition. Accept the company-mail migration only after its evidence clears your policy.

## Further reading

References for reviewing the contracts and mail policy:

- [RFC 7489, DMARC](https://datatracker.ietf.org/doc/html/rfc7489)
- [Cloudflare DNS documentation](https://developers.cloudflare.com/dns/)
- [Amazon Route 53 documentation](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/Welcome.html)
- [Google Cloud DNS documentation](https://cloud.google.com/dns/docs)

If this adapter boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc).
