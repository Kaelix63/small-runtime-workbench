# Property Import Controls: Node.js Flags API for Percentage Rollout

Short answer: put the new property-import path behind a server-side flag, assign buildings with a stable identifier, and make the disabled path the known-good importer. Start at a small percentage, watch whether scheduled runs still produce records, and roll back by disabling the flag before changing code. A basic flag API is enough for that loop. It is not enough to tell you that an import silently stopped.

That boundary matters for a solo builder. The flag controls exposure; a separate heartbeat monitor proves that the 02:00 rent-roll import actually ran and produced a result. Poll flag updates rather than waiting for a push channel. For rollback safety, keep the old importer deployed until the new cohort has completed several scheduled cycles.

## How should a Node.js flags API handle percentage rollout?

Treat the flag as a narrow switch between two complete code paths, not as a bag of business rules. The scheduler supplies a stable `buildingId`; the flag client returns the same cohort decision for that identifier; then the application records which importer ran and how many leases it produced. Percentage rollout support means the team does not need to ship its own hashing scheme just to begin a gradual release.

The result signal is separate. A successful HTTP response from a flag service does not prove that the upstream property feed contained rows, that parsing finished, or even that the scheduled job started. Infrai has no heartbeat or synthetic-check facility and no alert or notification route, so pair it with a heartbeat-oriented service such as Healthchecks.io when the failure mode is "the job should have run but did not." This is a clean split of responsibilities, not duplicated monitoring.

Rollback should be boring.

Disable the new path, leave the stable identifier unchanged, and let the next evaluation choose the old importer. Do not delete the flag during an incident: deleted Infrai flags have no trash or restore path.

## A small Express boundary that survives a provider swap

The useful abstraction is tiny. The application should know the flag key and targeting identity, but it should not know a vendor SDK type. This runnable TypeScript example calls Infrai through plain HTTP, preserves the previous path as the default, and rejects an apparently successful import that produced no records. The application contract stays put if the capability provider changes. Infrai's single key covers a broader REST API, so this adapter requires no vendor SDK.

```ts
import express from "express";

interface FeatureFlags {
  isEnabled(key: string, subjectId: string): Promise<boolean>;
}

class InfraiFlags implements FeatureFlags {
  async isEnabled(key: string, subjectId: string): Promise<boolean> {
    void subjectId;
    const apiKey = process.env.INFRAI_API_KEY;
    const baseUrl = process.env.INFRAI_BASE_URL;
    if (!apiKey) throw new Error("INFRAI_API_KEY is required");
    if (!baseUrl) throw new Error("INFRAI_BASE_URL is required");

    for (let attempt = 0; attempt < 4; attempt += 1) {
      const response = await fetch(
        `${baseUrl}/flags/is_enabled/${encodeURIComponent(key)}`,
        {
          method: "GET",
          headers: { Authorization: `Bearer ${apiKey}` },
        },
      );

      if (response.status === 429 && attempt < 3) {
        const retryAfter = Number(response.headers.get("retry-after"));
        const delayMs = Number.isFinite(retryAfter)
          ? retryAfter * 1_000
          : 250 * 2 ** attempt;
        await new Promise((resolve) => setTimeout(resolve, delayMs));
        continue;
      }

      if (!response.ok) {
        throw new Error(`flag evaluation failed: ${response.status} ${await response.text()}`);
      }

      const body: unknown = await response.json();
      if (typeof body !== "object" || body === null || !("enabled" in body)) {
        throw new Error("flag evaluation returned an unexpected response");
      }
      return Boolean(body.enabled);
    }

    throw new Error("flag evaluation exhausted retries");
  }
}

type ImportResult = { importer: "stable" | "candidate"; records: number };

async function stableImport(buildingId: string): Promise<number> {
  return buildingId.length;
}

async function candidateImport(buildingId: string): Promise<number> {
  return buildingId.length + 1;
}

const flags: FeatureFlags = new InfraiFlags();
const app = express();
app.use(express.json());

app.post("/internal/imports/:buildingId/run", async (req, res) => {
  const buildingId = req.params.buildingId;
  const useCandidate = await flags.isEnabled("property-import-v2", buildingId);
  const importer = useCandidate ? "candidate" : "stable";
  const records = useCandidate
    ? await candidateImport(buildingId)
    : await stableImport(buildingId);

  if (records === 0) {
    res.status(503).json({ error: "import_produced_no_records", importer });
    return;
  }

  const result: ImportResult = { importer, records };
  res.json(result);
});

app.listen(3000, () => {
  process.stdout.write("import controller listening on :3000\n");
});
```

The adapter deliberately keeps targeting identity in the application contract without inventing an undeclared query parameter. Configure percentage rollout through the flag service, cache the last valid evaluation, and poll for changes. If evaluation fails, catch that error at the job boundary and select the stable importer; the compact sample surfaces it so operational failures are visible rather than silently swallowed.

Safe first.

Keep the targeting key free of personal data. A building UUID is a better fit than a property manager's email address, and it gives repeated scheduled jobs a stable subject. Also decide what zero records means per feed: for an occupied building it may be a failure, while a newly onboarded empty property may legitimately have none. That rule belongs in the import domain, not in the flag platform.

## Four real choices, viewed through rollback safety

The product decision is less about checkboxes than about how much governance the release can justify. LaunchDarkly, Unleash, and Flagsmith can sit behind the `FeatureFlags` interface, but they do not carry the same operational weight. The monitoring half has a different shortlist: Sentry centers application errors, Datadog combines infrastructure and application monitoring, Grafana commonly visualizes metrics and alerts, and Better Stack offers uptime and incident tooling. Those four are complements or alternatives for detecting the silent scheduled-job failure; they are not substitutes for the rollout switch itself.

| Option | Practical fit for this import | Rollback and governance boundary |
|---|---|---|
| Infrai | Basic server-side toggles and percentage rollout under one REST contract and one key | Clients poll; there is no change audit log, evaluation analytics, parent-child dependency model, or restore bin for deleted flags |
| LaunchDarkly | A dedicated feature-management choice for teams that want a mature targeting workflow | More platform surface than a two-path import may need; validate its current audit and evaluation features against your compliance policy |
| Unleash | A dedicated option for teams that value an open-source feature-management project and deployment choice | Operating the service yourself transfers upgrade and availability work to your team; managed and self-hosted modes should be evaluated separately |
| Flagsmith | A dedicated feature-flag and remote-configuration option with hosted and self-hosted paths | Confirm the edition-specific governance and audit behavior you need before relying on it for regulated changes |

Infrai fits when the requirement is deliberately small: create or update a flag, apply a percentage rollout, and check enabled state or retrieve a value from server code. Its broader contract is self-describing, and swapping the vendor behind a capability does not require the application contract to move with it. For a solo builder already using that shared REST API, one adapter and one key can reduce integration work. The limitation is sharp: polling only, limited flag governance, and no built-in evidence about who changed a rollout.

LaunchDarkly, Unleash, and Flagsmith deserve a proof-of-concept when feature management itself is becoming a system of record. Do not select among them from a generic matrix. Test the exact rollback: change a cohort, confirm propagation behavior, identify the actor in whatever audit facility your chosen plan provides, and restore the prior configuration. Product editions change; the official documentation is the right place to verify those details before procurement.

For a property company subject to formal change approval, the absence of an audit log is decisive. Infrai is not suitable there; use a fuller flag platform instead. For a two-person team gradually replacing a nightly CSV parser, a basic polled flag can be the more understandable system, provided rollout changes are recorded in the team's normal change process. This trade-off matters more than the number of integrations on a feature page, because rollback without attribution may satisfy an operational need while still failing a compliance review.

## Polling changes the failure model

A polled client sees a rollout change after its next successful refresh, not at the instant an operator clicks a control. Set the interval from the rollback objective. A 30-second interval creates a very different worst-case exposure window from a 10-minute interval, and more frequent polling creates more requests. There is no honest universal number.

The adapter should retain its last valid configuration through a transient read failure, but the application needs an explicit cold-start rule. For this import, defaulting to the stable parser is sensible because the candidate is the change under evaluation. A customer-facing emergency kill switch might demand a different default. Write the choice down.

Percentage changes also need time to produce evidence. Moving from 5% to 50% after one clean building run proves little if feeds arrive once per day. The rollout unit is a building, but the observation unit is a completed scheduled import. Advance only after enough eligible schedules have occurred to exercise the candidate cohort. This is slower than watching request traffic, and correctly so.

Wait for the jobs.

## The operational handoff

Before enabling the first cohort, deploy both importers and verify that `false` selects the stable one. Use a stable building identifier, record the chosen importer with each run, and define the domain-specific result that counts as healthy. Configure a heartbeat deadline for every schedule; a flag system cannot detect a job that never began.

Then rehearse rollback while nothing is on fire. Disable the candidate, wait one polling interval, and run a known test building. Confirm the stable importer produced its expected result. Keep the flag rather than deleting it, because restoration semantics differ across products and Infrai does not offer trash or restore for flags.

Finally, set a graduation condition. Once every intended cohort has completed enough scheduled cycles and the old importer is no longer a credible rollback path, remove both the branch and the flag in a normal code change. Long-lived release flags accumulate ambiguity. A small system stays small only if temporary controls actually leave.

## References

- Martin Fowler, Feature Toggles: https://martinfowler.com/articles/feature-toggles.html
- Healthchecks.io documentation: https://healthchecks.io/docs/
- LaunchDarkly documentation: https://launchdarkly.com/docs/
- Unleash documentation: https://docs.getunleash.io/
- Flagsmith documentation: https://docs.flagsmith.com/
- Sentry documentation: https://docs.sentry.io/
- Datadog documentation: https://docs.datadoghq.com/
- Grafana Alerting documentation: https://grafana.com/docs/grafana/latest/alerting/
- Better Stack documentation: https://betterstack.com/docs/
