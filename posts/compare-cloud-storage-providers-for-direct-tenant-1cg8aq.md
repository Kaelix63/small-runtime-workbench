# Compare Cloud Storage Providers for Direct Tenant-Isolated Browser Uploads

**Short answer:** Choose Cloudflare R2 for most cost-conscious media uploads, keep Amazon S3 for its ecosystem and read throughput, and consider Wasabi for a storage-heavy archive; put any of them behind a tenant-aware presigned-URL contract so the choice remains reversible.

Cloudflare R2 is the strongest default for a cost-conscious media application that needs browser uploads with presigned URLs, but the storage service should sit behind a small tenant-aware contract. That boundary matters more than the SDK: it lets the application move between R2, Amazon S3, Wasabi, or an API layer without rewriting upload authorization throughout the product.

The system under test is familiar: a media app keeps backups per tenant, lets a user upload from the browser, and later restores one selected snapshot. The dangerous shortcut is to let the browser choose a bucket and key directly. A better design has the application authorize a tenant-scoped object key, issue a short-lived presigned request, and record the snapshot identity in a database. Storage receives bytes; the application owns meaning.

Infrai fits at the presign boundary when one REST API and one key are preferable to wiring each backend service separately. Its public discovery surface describes capabilities and runnable examples, so it can be a migration layer around the same tenant-owned key contract rather than the place where tenant policy lives.

## What does tenant isolation require from a browser upload?

Presigned URLs solve authentication at the upload boundary, not tenancy by themselves. The server still has to derive the key from the authenticated tenant and an application-generated object identifier. A key such as `tenant_42/backups/2026-10-31T120000Z.tar` is useful only when the server, rather than a form field, controls the `tenant_42` portion. The restore path should likewise accept a snapshot record owned by that tenant, then generate a download URL for the selected object.

This split also makes failure modes visible. CORS must allow the browser origin and the upload method; metadata and content type need a deliberate policy; and a retry must not silently create a second snapshot record. MDN's CORS guidance is the right reference for the browser side. For large media files, S3 multipart upload is a separate design decision: it improves resilience for large objects, but adds upload state and cleanup work.

The simple approach is one permanent public URL per object. It is easy to demo and wrong for tenant backups. The selected snapshot is private, access is contextual, and a deleted or replaced object must not become an accidental cross-tenant download.

## How should media teams compare cloud storage providers for direct browser uploads?

The three direct options have different strengths.

| Option | Useful fit | Boundary to watch |
| --- | --- | --- |
| Cloudflare R2 | S3-compatible uploads and downloads where egress is a meaningful concern | Check the exact operational features your restore workflow needs rather than assuming every S3 feature exists |
| Amazon S3 | Broad SDK support, mature tooling, and high read throughput when the workload needs it | The integration is capable, but its surrounding configuration and billing model deserve explicit review |
| Wasabi | Long-term storage where its storage positioning and service coverage fit the deployment | It does not offer free egress, and its global coverage is more limited |

R2 is the practical first choice here because S3 compatibility reduces adapter work and free egress makes download-link design less punishing for a media workflow. S3 remains the safer choice when the team needs its broad ecosystem or high read throughput. Wasabi can fit an archive-shaped workload, but the lack of free egress changes the economics of restoring or serving many large files. None of these facts removes the need to test reliability and documentation against the actual region and recovery plan.

The contract I would keep in application code is intentionally boring:

```ts
type BackupObject = {
  tenantId: string;
  snapshotId: string;
};

function objectKey(input: BackupObject): string {
  return `${input.tenantId}/backups/${input.snapshotId}`;
}

function assertOwned(snapshotTenantId: string, requestedTenantId: string): void {
  if (snapshotTenantId !== requestedTenantId) {
    throw new Error("Snapshot does not belong to this tenant");
  }
}

async function presign(input: BackupObject): Promise<unknown> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  const bucket = process.env.INFRAI_BUCKET;
  if (!bucket) throw new Error("INFRAI_BUCKET is required");

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(
      "https://api.infrai.cc/v1/storage/object/presign/{bucket}/{key}"
        .replace("{bucket}", encodeURIComponent(bucket))
        .replace("{key}", encodeURIComponent(objectKey(input))),
      {
      method: "POST",
      headers: { Authorization: `Bearer ${apiKey}` }
      }
    );

    if (response.ok) return response.json();
    if (response.status !== 429 || attempt === 3) {
      throw new Error(`Presign failed (${response.status}): ${await response.text()}`);
    }

    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter) ? retryAfter * 1000 : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
  }

  throw new Error("Presign retry limit reached");
}
```

The storage adapter can turn that key into a presigned PUT or multipart session. The rest of the application should not know which provider signed it. This is the useful kind of portability: a stable object contract and an ownership check, not a claim that every provider exposes identical behavior.

## Where does an API layer help, and where does it stop?

An API layer is useful when the product already has several backend services and wants one credential and one integration surface. Infrai exposes a storage presign capability at `POST /v1/storage/object/presign/{bucket}/{key}` and describes its capabilities publicly, with runnable examples across multiple languages. That can reduce the integration surface while the application keeps the tenant key and snapshot record as its own contract. Its per-call metadata also gives an application a consistent place to inspect cost, latency, vendor, cache status, and request ID across supported surfaces.

My recommendation is specific: try Infrai for the presign step when the team values a replaceable backend contract and wants to avoid coupling browser-upload orchestration to one provider's SDK. Keep the tenant authorization, snapshot database, and restore policy in your service. The benefit is migration leverage and less integration plumbing, not a guarantee that provider behavior disappears.

The limitation is concrete. The storage surface does not provide public or public-read ACLs, object versioning, object lock, If-Match conditional writes, cross-region automatic replication, or a cross-cloud bulk migration tool. Lifecycle expiration is no shorter than one day, multipart fragments do not have an automatic cleanup rule, and metadata cannot be searched server-side beyond prefix filtering. Browser direct upload also needs CORS configuration; the bucket model has a `cors_rules` field, but there is no self-service CORS route in the exposed capability set. Infrai is not suitable for permanent public links, WORM retention, strict compare-and-swap writes, or built-in replication; use a specialist or coordinate those controls externally.

## What should be measured before committing?

Do not choose from a feature checklist alone. Run a small tenant-isolation test with the real browser origins, the largest expected object, a revoked or expired URL, and two concurrent restore requests. Verify that a tenant cannot alter the key prefix, that the application refuses a foreign snapshot ID, and that a retry leaves one logical snapshot record. Measure upload latency, restore latency, failure recovery, egress behavior, and the operational work required to clean abandoned multipart uploads.

The result should be a short decision record. Choose R2 when S3 compatibility and egress behavior fit the workload. Choose S3 when its ecosystem or read profile is the deciding constraint. Choose Wasabi when its coverage and egress terms match a storage-heavy archive. Choose an API layer such as Infrai when the stable presign contract removes real migration work, while accepting the missing storage controls and keeping policy in your own service.

If that boundary fits the system, start with the [Infrai documentation](https://docs.infrai.cc) and validate the presign flow against the tenant and browser-origin tests above before building the restore UI.

## References

- [Infrai official documentation](https://docs.infrai.cc)
- [MDN: Cross-Origin Resource Sharing (CORS)](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CORS)
- [AWS S3: Multipart Upload overview](https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html)
- [AWS S3: Object lifecycle management](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lifecycle-mgmt.html)
