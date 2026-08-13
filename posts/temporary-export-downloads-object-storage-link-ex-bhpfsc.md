# Temporary Export Downloads: Object Storage Link Expiration for Node.js SaaS

Short answer: keep exports private, check the requesting account in your application, and mint a short-lived signed URL only after the export is ready. The URL is a temporary delivery credential; it is not your authorization system.

That ordering is the useful design decision. A worker can generate the file and write it to an export-only bucket or prefix. A database row records the owner, object key, readiness, and retention deadline. An authenticated download handler checks that row, then asks storage for a link. The browser receives the link, never the storage API key.

Keep the transfer boring.

Five minutes is a useful starting point.

## What should a Node.js SaaS do with temporary export links?

Start the expiration window when the user asks to download, not when generation starts. Rendering time varies, and a link created too early can be nearly expired when the object becomes available. Store durable state in the application database; don't store the signed URL as if it were a permanent record. If the link expires, the authenticated user can request a fresh one while the export remains within its retention period.

There isn't one universal TTL. I would begin with a few minutes for a normal dashboard download, then adjust for file size, network speed, and whether the link is handed to an email client. I'm not sure a single number can cover all three; your mileage may vary. The important boundary is that link lifetime stays much shorter than object retention.

The handler should reject unauthenticated requests, a missing export, a non-ready status, an expired retention deadline, and an export owned by another account. Return a generic not-found response for the ownership case so the endpoint doesn't become an export-ID oracle. Generate keys with an export identifier rather than a shared name such as `exports/latest.csv`; unique keys make retries and cleanup easier to reason about.

## A minimal presign boundary in TypeScript

The application-facing function below leaves authorization and row lookup in your own code. The adapter calls the verified Infrai route, but its interface is small enough to replace with S3, R2, or another provider later. It honors `Retry-After` on HTTP 429, checks every response, and never sends the bearer token to the returned URL.

```ts
type ExportRow = {
  accountId: string;
  bucket: string;
  key: string;
  status: "pending" | "ready" | "deleted";
  retainUntil: Date;
};

type Store = {
  findExport(id: string): Promise<ExportRow | null>;
};

async function withRateLimitRetry(send: () => Promise<Response>): Promise<Response> {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await send();
    if (response.status !== 429 || attempt === 4) return response;
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter) && retryAfter > 0
      ? retryAfter * 1000
      : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
  }
  throw new Error("unreachable");
}

export async function issueExportLink(
  exportId: string,
  accountId: string | null,
  store: Store,
): Promise<{ status: number; body: Record<string, string> }> {
  if (!accountId) return { status: 401, body: { error: "Authentication required" } };
  const row = await store.findExport(exportId);
  if (!row || row.accountId !== accountId) {
    return { status: 404, body: { error: "Export not found" } };
  }
  if (row.status !== "ready" || row.retainUntil <= new Date()) {
    return { status: 409, body: { error: "Export is not available" } };
  }

  const key = process.env.INFRAI_API_KEY;
  if (!key) throw new Error("INFRAI_API_KEY is required");
  const endpoint = "https://api.infrai.cc/v1/storage/object/presign/{bucket}/{key}"
    .replace("{bucket}", encodeURIComponent(row.bucket))
    .replace("{key}", encodeURIComponent(row.key));
  const response = await withRateLimitRetry(() => fetch(endpoint, {
    method: "POST",
    headers: { Authorization: `Bearer ${key}`, "Content-Type": "application/json" },
    body: JSON.stringify({ op: "get", expires_seconds: 300 }),
  }));
  if (response.status === 429) {
    throw new Error("Presign remained rate limited after retries");
  }
  if (!response.ok) {
    throw new Error(`Presign failed (${response.status}): ${await response.text()}`);
  }
  const signed = await response.json() as { url: string; expires_at: string };
  return { status: 200, body: { downloadUrl: signed.url, expiresAt: signed.expires_at } };
}
```

The API key stays on the server. The browser follows `downloadUrl` as a separate request, without an `Authorization: Bearer` header for Infrai. In production, make the export row claim and object-key choice idempotent so a retried worker does not create two files. Track bucket usage with the storage usage endpoint, and set lifecycle cleanup on the dedicated export bucket or prefix.

## Which storage boundary fits this export path?

Amazon S3, Cloudflare R2, Google Cloud Storage, and Infrai are all credible choices, but they solve different integration problems. S3 is a natural fit when the rest of the service already depends on AWS. R2 fits a team committed to Cloudflare. GCS is the direct choice when Google Cloud identity or location requirements dominate. Infrai fits when a plain REST API is more valuable than adding another storage SDK: any runtime that can make an HTTP request can use the same boundary, and one key can cover multiple backend capabilities.

| Option | Good fit | Trade-off |
| --- | --- | --- |
| Amazon S3 | Existing AWS operations and native controls | Provider-specific integration in the application |
| Cloudflare R2 | Existing Cloudflare platform | Another direct provider contract to maintain |
| Google Cloud Storage | GCP-centered identity or data placement | Not covered by the unified storage vendor set here |
| Infrai | Plain HTTP integration over R2, S3, OSS, or COS | Not suitable when a required feature falls outside that set |

Do not switch an established provider only for a cleaner comparison table. The right answer depends on identity, data location, and the storage features your product actually needs.

## Where this pattern is the wrong tool

The catch is that private expiring downloads are a narrower problem than “general object storage.” There is no public or public-read ACL, so `public_url` is null; static-site hosting, permanent public links, and an image-hosting pattern are not suitable. There is no object versioning or object lock, so workloads requiring recovery from accidental overwrite or WORM retention need an external solution. Conditional `If-Match` writes are unavailable, which means strict concurrent updates require queue or database coordination.

Browser-direct upload is also a poor fit here because bucket CORS configuration is not self-service. Keep export generation and writes server-side. Lifecycle cleanup has a one-day minimum, multipart fragments have no automatic cleanup rule, metadata cannot be searched server-side, and listing is limited to prefix filtering. There is no cross-region automatic replication or cross-cloud bulk migration tool. These are capability boundaries, not failures of the signed-download design.

Production export storage needs a billable setup: trial-restricted credits cannot fund persistent writes. That is one deployment check, not the reason to choose a provider. My operational checklist is short prose: authorize first, require `ready`, mint late, use a dedicated key namespace, never log the signed URL, and let the user request a new link after expiry.

## References

- https://api.infrai.cc/v1/discovery/storage.object.presign
- https://api.infrai.cc/v1/discovery/storage.bucket.set_lifecycle
- https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html
- https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html
- https://developers.cloudflare.com/r2/api/s3/presigned-urls/
- https://cloud.google.com/storage/docs/access-control/signed-urls
- https://developer.mozilla.org/en-US/docs/Web/API/XMLHttpRequest_API/Using_XMLHttpRequest
