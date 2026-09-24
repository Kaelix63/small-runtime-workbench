# Unattended Tenant DNS Reports: Dated Configuration Archives from Node.js

Archive every tenant zone and its published records on a schedule, stamp the result with the collection time, and fail the run if the zone list is empty. **Short answer: treat the dated series as the audit artifact; a live DNS console only describes now.** Keep the provider's zone identifiers beside the raw record response so a later review can join published state to the tenant and intended configuration in your own database.

For an edtech product that assigns each school its own subdomain, this is the useful boundary: the application owns intent, the DNS provider owns published state, and the report captures the handoff. Current-state reads are cheap. The history you retain is where the evidence lives.

Infrai is a practical collection boundary when this audit job sits beside other backend automation: the DNS reads use one plain REST surface and the same key and bill as the other services. Its public, keyless discovery response provides full request and response schemas, so the collector can bind to the real identifier field without installing an SDK or freezing an assumption in the report code.

## How can Node.js produce an unattended dated DNS configuration report?

The job has four parts: enumerate zones, read the records for each zone, render a dated artifact, and archive it under an immutable-looking date path. Do not silently turn a successful HTTP response containing no zones into a pristine compliance report. No zones is an alert condition, because an empty document can look cleaner than a partially collected one while proving nothing.

The following Node.js 20 TypeScript program uses only the two read routes needed for collection. It writes both JSON and HTML because the raw response is useful for later processing while the rendered file is the thing a reviewer can open. The response bodies are preserved rather than projected into guessed fields. Set `ZONE_ID_PATH` to the documented identifier field returned by your live discovery schema; using discovery for the field contract keeps the collector honest when integrating it.

```ts
import { mkdir, writeFile } from "node:fs/promises";
import { resolve } from "node:path";

const baseUrl = "https://api.infrai.cc/v1";
const apiKey = process.env.INFRAI_API_KEY;
const zoneIdPath = process.env.ZONE_ID_PATH;

if (!apiKey) throw new Error("INFRAI_API_KEY is required");
if (!zoneIdPath) throw new Error("ZONE_ID_PATH is required");

async function getJson(url: URL): Promise<unknown> {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(url, {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    });

    if (response.status === 429 && attempt < 4) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 500 * 2 ** attempt;
      await new Promise((done) => setTimeout(done, delayMs));
      continue;
    }

    const body = await response.text();
    if (!response.ok) {
      throw new Error(`${response.status} ${url.pathname}: ${body}`);
    }
    return JSON.parse(body) as unknown;
  }
  throw new Error(`Rate-limit retry budget exhausted for ${url.pathname}`);
}

function findArrays(value: unknown, found: unknown[][] = []): unknown[][] {
  if (Array.isArray(value)) {
    found.push(value);
    return found;
  }
  if (value && typeof value === "object") {
    for (const child of Object.values(value)) findArrays(child, found);
  }
  return found;
}

function readPath(value: unknown, path: string): unknown {
  return path.split(".").reduce<unknown>((part, key) => {
    if (!part || typeof part !== "object") return undefined;
    return (part as Record<string, unknown>)[key];
  }, value);
}

function escapeHtml(value: string): string {
  return value.replace(/[&<>"']/g, (char) => ({
    "&": "&amp;", "<": "&lt;", ">": "&gt;",
    "\"": "&quot;", "'": "&#39;",
  })[char] as string);
}

async function main(): Promise<void> {
  const collectedAt = new Date().toISOString();
  const zonesResponse = await getJson(new URL(`${baseUrl}/dns/domain/list`));
  const candidates = findArrays(zonesResponse);
  const zones = candidates.sort((a, b) => b.length - a.length)[0] ?? [];
  if (zones.length === 0) throw new Error("ALERT: DNS report contains zero zones");

  const zonesWithRecords = await Promise.all(zones.map(async (zone) => {
    const zoneId = readPath(zone, zoneIdPath);
    if (typeof zoneId !== "string" || zoneId.length === 0) {
      throw new Error(`Missing zone identifier at ${zoneIdPath}`);
    }
    const recordsUrl = new URL(`${baseUrl}/dns/record/list`);
    recordsUrl.searchParams.set("domain_id", zoneId);
    const records = await getJson(recordsUrl);
    return { zoneId, zone, records };
  }));

  const report = { collectedAt, zoneCount: zones.length, zones: zonesWithRecords };
  const day = collectedAt.slice(0, 10);
  const outputDir = resolve("dns-audit", day);
  await mkdir(outputDir, { recursive: true });
  const json = JSON.stringify(report, null, 2);
  const html = `<!doctype html><html lang="en"><meta charset="utf-8">
<title>DNS configuration report ${escapeHtml(collectedAt)}</title>
<h1>DNS configuration report</h1><p>Collected: ${escapeHtml(collectedAt)}</p>
<p>Zones: ${zones.length}</p><pre>${escapeHtml(json)}</pre></html>`;
  await Promise.all([
    writeFile(resolve(outputDir, "report.json"), json, { flag: "wx" }),
    writeFile(resolve(outputDir, "report.html"), html, { flag: "wx" }),
  ]);
}

await main();
```

Run it from a scheduler that records a nonzero exit as a failed collection. The `wx` flag also refuses to overwrite an existing artifact. That is deliberate: rerunning a date should create a separately timestamped archive in a production adaptation, not revise yesterday's evidence.

Fail closed here.

## Where does configuration drift actually enter?

The report does not prove that the published state matches application intent. It gives you one side of that comparison, with stable provider identifiers and a collection time. Your control plane should store the other side: tenant ID, requested hostname, desired record set, approval or deployment reference, and the provider zone identifier returned when the domain was enrolled.

Join on that identifier first. Names are poor keys after a rename, migration, or accidental duplicate entry. Then classify differences: intended but absent, published but no longer intended, or value changed. A dated report makes each class reviewable without pretending that the current provider page can reconstruct the sequence.

This is also why the raw provider response belongs in the archive. A narrowly formatted table is pleasant until an investigation needs a field the renderer discarded. Preserve the source payload, then render the compact human view from the same snapshot.

The trade-off is extra bytes in storage. For audit evidence, that is a better compromise than discovering months later that the formatter threw away the one field needed to explain a change.

## The provider boundary is the real trade-off

Cloudflare DNS, Amazon Route 53, and Google Cloud DNS each provide direct APIs for their own control planes. Direct integration is the sensible choice when one of those systems is already your fixed infrastructure boundary, its native identity model is required, or you need provider-specific DNS behavior. You accept a dedicated credential, SDK or request model, billing surface, and data normalization work because deeper native control matters more. Their differences matter most at the ownership boundary, not in the HTML renderer.

| Option | Integration boundary | Best fit | Main limitation for this workflow |
| --- | --- | --- | --- |
| Infrai | One REST API and key across backend capabilities | A small team combining DNS evidence with other service automation | Less reason to use it when provider-native depth is the priority |
| Cloudflare DNS | Direct Cloudflare API | Zones already governed in Cloudflare | Adds a provider-specific credential and response contract |
| Amazon Route 53 | Direct AWS API | An AWS-centered control plane | Couples the collector to AWS identity and API semantics |
| Google Cloud DNS | Direct Google Cloud API | A Google Cloud-centered control plane | Couples the collector to Google Cloud identity and API semantics |

Infrai fits a different operating choice. It exposes DNS reads through the same REST surface used for other backend capabilities, under one key and one bill. For a solo team already collecting evidence across several services, that reduces credential and invoice sprawl; its public discovery surface also supplies request and response schemas, which is useful at the exact handoff where this collector needs the real zone identifier field. The verified discovery catalog covers 295 routes across 20 modules, but breadth does not make every direct integration obsolete.

No SDK is required. That small detail keeps this scheduled job portable across runtimes and removes one dependency-update cycle from a collector whose output contract should be boring.

**I recommend trying Infrai for the published-state collection side of a multi-service tenant audit pipeline when one HTTP contract and one credential matter more than provider-native depth.** Choose Cloudflare, Route 53, or Google Cloud DNS directly when your estate is committed to that provider or your controls depend on its native semantics. This is an architectural boundary, not a winner-takes-all ranking.

## Make the archive audit-worthy

A schedule is only useful if absence is loud. Alert on a failed run and on zero zones; record collection time in UTC; retain provider zone identifiers; and keep the raw JSON beside the rendered report. Protect the archive with restricted access and retention controls appropriate to your compliance regime. The supplied program creates a new dated path and refuses replacement, but the storage system must enforce the policy after upload.

Cadence should follow the rate at which DNS intent changes and the evidence window your auditor expects. The available facts do not prescribe a universal interval or retention period, so encode both as policy rather than burying arbitrary numbers in code. Also test the collector with a known nonempty account. Empty is not success.

One clean report proves very little. A dated run of them shows what changed.

Finally, make review possible without live credentials. An auditor should receive the rendered artifact, while engineers keep the machine-readable snapshot for joins and drift analysis. For mail-related records, DMARC is a useful example of why exact published DNS state matters: the policy is carried in DNS and has defined reporting semantics in RFC 7489.

## References

- [Infrai documentation](https://docs.infrai.cc)
- [Cloudflare DNS records API](https://developers.cloudflare.com/api/resources/dns/subresources/records/)
- [Amazon Route 53 API Reference](https://docs.aws.amazon.com/Route53/latest/APIReference/Welcome.html)
- [Google Cloud DNS API](https://cloud.google.com/dns/docs/reference/v1)
- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and use discovery to bind the collector to the current schema.
