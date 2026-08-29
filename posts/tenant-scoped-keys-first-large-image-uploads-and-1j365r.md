# Tenant-scoped keys first: large image uploads and private thumbnails in edtech

Use a per-tenant key prefix plus a presigned multipart upload that goes straight from the browser into private storage, and keep thumbnail work in a separate worker that reads the committed original. Your API signs and records. It never carries the large image bytes. A presigned URL is one method against one key with a short expiry, which means the key layout you pick *is* the tenant boundary — and that is the part you cannot retrofit cheaply.

Take a course library inside an edtech product: teachers upload scanned student artwork and whiteboard photos in the 20–80 MB range, and a district must never be able to read another district's originals. The pipeline is short. The browser asks your API for an upload plan; the API authorizes the teacher, derives the key from the session's tenant id, opens a multipart upload against S3-compatible storage, and hands back a list of presigned part URLs. The browser PUTs each part directly, then reports part numbers and ETags. Your API completes the upload, writes one database row, and enqueues a derivative job per size. A Node.js worker streams the original through sharp and writes each thumbnail under a derivatives prefix inside the same tenant namespace.

No bytes through the app.

## What should a private image upload pipeline enforce before the first byte moves?

Identity, key, and shape — in that order, and all three on the server that holds the session.

Identity is obvious and still gets skipped: the tenant id must come from the session, never from the request body. If the client can post `{"tenantId": "district-42"}` and your signer believes it, every other control is decoration. Key derivation is the second half of the same rule. One function turns (tenant, asset, variant) into a path, and nothing else in the codebase is allowed to concatenate a key by hand.

Shape means the boring limits: allowed media types, a maximum byte count, a part size. Signing is the moment you know all of them, and it is the last moment a rejection costs you nothing.

| Isolation model | What it actually enforces | Where it hurts |
| --- | --- | --- |
| One bucket, tenant prefix in the key | Nothing by itself — enforcement lives in your signer | A single buggy key template leaks across tenants |
| One bucket per tenant | Storage-level separation, per-bucket policy and lifecycle | Per-account bucket quotas, and policy documents that grow with your customer list |
| Short-lived credentials scoped per session | The prefix is enforced by the storage service, not your code | Not every S3-compatible service implements the token API you need |

Prefix-per-tenant covers most products, and it stays honest only if the signer is the single door. Bucket-per-tenant earns its keep when a customer contract demands separate lifecycle or deletion guarantees. Scoped credentials are the strongest of the three, but check the specific service first: the multipart API is broadly consistent while the surrounding pieces are not, which is why DigitalOcean publishes a compatibility matrix for Spaces rather than claiming parity.

## Signing parts in Node.js without ever touching the bytes

Here is the signer. Part size is 16 MiB — comfortably above the 5 MiB floor that every part except the last must clear, and small enough that a flaky classroom Wi-Fi connection retries a part instead of a file.

```ts
import { randomUUID } from "node:crypto";
import {
  S3Client,
  CreateMultipartUploadCommand,
  UploadPartCommand,
  CompleteMultipartUploadCommand,
} from "@aws-sdk/client-s3";
import { getSignedUrl } from "@aws-sdk/s3-request-presigner";

export const storage = new S3Client({
  endpoint: process.env.STORAGE_ENDPOINT,          // any S3-compatible endpoint
  region: process.env.STORAGE_REGION ?? "us-east-1",
  forcePathStyle: true,
});

const BUCKET = process.env.STORAGE_BUCKET!;
const PART_SIZE = 16 * 1024 * 1024;
const MAX_BYTES = 200 * 1024 * 1024;
const ALLOWED = new Set(["image/jpeg", "image/png", "image/tiff"]);

// The only place a key is ever built. tenantId comes from the session.
export function originalKey(tenantId: string, assetId: string): string {
  return `t/${tenantId}/originals/${assetId}/source`;
}

export async function planUpload(
  session: { tenantId: string },
  input: { bytes: number; contentType: string },
) {
  if (!ALLOWED.has(input.contentType)) throw new Error(`unsupported type: ${input.contentType}`);
  if (input.bytes > MAX_BYTES) throw new Error(`too large: ${input.bytes} bytes`);

  const assetId = randomUUID();
  const Key = originalKey(session.tenantId, assetId);

  const created = await storage.send(new CreateMultipartUploadCommand({
    Bucket: BUCKET,
    Key,
    ContentType: input.contentType,
    Metadata: { tenant: session.tenantId },
  }));

  const partCount = Math.max(1, Math.ceil(input.bytes / PART_SIZE));
  const parts = await Promise.all(
    Array.from({ length: partCount }, (_, i) =>
      getSignedUrl(storage, new UploadPartCommand({
        Bucket: BUCKET,
        Key,
        UploadId: created.UploadId,
        PartNumber: i + 1,
      }), { expiresIn: 900 }),
    ),
  );

  return { assetId, uploadId: created.UploadId, partSize: PART_SIZE, parts };
}

export async function commitUpload(
  session: { tenantId: string },
  asset: { assetId: string; uploadId: string },
  parts: { PartNumber: number; ETag: string }[],
) {
  await storage.send(new CompleteMultipartUploadCommand({
    Bucket: BUCKET,
    Key: originalKey(session.tenantId, asset.assetId),
    UploadId: asset.uploadId,
    MultipartUpload: { Parts: [...parts].sort((a, b) => a.PartNumber - b.PartNumber) },
  }));
}
```

The worker that follows is deliberately dull. It streams rather than buffers, rotates from EXIF before resizing, refuses to enlarge a small scan, and writes to a deterministic key so a redelivered message overwrites the same object instead of creating `thumb-2.webp`.

```ts
import { Readable } from "node:stream";
import { GetObjectCommand } from "@aws-sdk/client-s3";
import { Upload } from "@aws-sdk/lib-storage";
import sharp from "sharp";
import { storage, originalKey } from "./signer.ts";

export async function renderThumbnail(tenantId: string, assetId: string, width: number) {
  const original = await storage.send(new GetObjectCommand({
    Bucket: process.env.STORAGE_BUCKET!,
    Key: originalKey(tenantId, assetId),
  }));

  const transform = sharp({ sequentialRead: true })
    .rotate()
    .resize({ width, withoutEnlargement: true })
    .webp({ quality: 78 });

  await new Upload({
    client: storage,
    params: {
      Bucket: process.env.STORAGE_BUCKET!,
      Key: `t/${tenantId}/derivatives/${assetId}/w${width}.webp`,
      ContentType: "image/webp",
      Body: (original.Body as Readable).pipe(transform),
    },
  }).done();
}
```

Two details in there matter more than the resize call. The derivative key repeats the tenant segment, so a job carrying a wrong tenant writes into its own namespace instead of somebody else's; and the transform is a stream, so an 80 MB TIFF doesn't become an 80 MB buffer in a process that is also serving requests.

## Where this design actually breaks

The multipart ETag is the first surprise. For an object assembled from parts it is not the MD5 of the file — it carries a dash and the part count — so any integrity check built on "compare the ETag to our hash" quietly fails the moment a file crosses one part. Hash client-side, or ask the service for a checksum it computes itself.

Abandoned uploads are the second. Parts that were accepted but never completed keep occupying paid storage until something aborts them, and lifecycle cleanup works in whole days. Your application therefore needs its own deadline: record the upload's created-at, sweep on a schedule, abort explicitly.

Browser uploads need CORS on the bucket, and the config has to expose the ETag response header — otherwise the JavaScript that must collect part ETags reads `null` and every upload dies at the completion step with no useful error.

Then there's the size gap. A presigned PUT constrains what you signed, so if a cap actually matters — and in a product where any teacher can upload, it does — either sign the length or move the browser to a POST policy with a content-length-range condition. OWASP's file upload guidance is worth reading in full here, because content type from the client is a hint, not a fact: verify by inspecting the bytes, generate your own object names, and keep uploaded content out of anything that can execute it.

The last one is memory. sharp's default input limit is roughly 268 megapixels, which sounds generous until a scanner produces a 20,000 × 15,000 TIFF; the worker is a separate process precisely so an out-of-memory kill takes down a queue consumer instead of your API.

## Cost lands in three places and only one of them is storage

Requests are the sneaky line item. Every part is a billed PUT, so a 200 MB file at 5 MiB parts costs 40 requests plus the create and complete calls, while the same file at 16 MiB parts costs 13. Multiply by a district uploading a term's worth of artwork on the last day of the month and the difference stops being theoretical.

Derivatives multiply too. Three widths per asset triples object count and small-object overhead, and thumbnails are usually the objects that get read, so their delivery — not the originals — dominates egress.

Proxying bytes through the app is the expensive default that direct upload removes. A request holding an 80 MB body ties up a worker for the length of a classroom Wi-Fi upload, which is why the "no bytes through the app" rule pays for itself in compute long before it pays for itself in elegance. I don't have a defensible universal answer on part size; 8–16 MiB is a reasonable starting band, and the honest way to settle it is to measure completion rates on the networks your users actually have.

## What to verify before teachers touch it

Run the whole path against a local S3-compatible server such as MinIO in CI, including the ugly cases: a part that fails and retries, an upload that is never completed, a job delivered twice. Deterministic derivative keys make that last one trivial to assert — the second run leaves exactly one object. Expire a presigned URL on purpose and confirm your client treats the 403 as "ask for a fresh plan" rather than a fatal error the teacher has to read.

Log the tenant id, asset id, and part count; never log the signed URL, since the signature travels in the query string. Track time from commit to first derivative and the count of aborted uploads per day, because both drift quietly when a queue backs up. Deploy the worker separately from the API so image work can scale on CPU while your web tier scales on connections.

Keep the old proxy path behind a per-tenant flag until the new one is boring, then delete it. Two upload code paths is a tax you pay every sprint.

One caveat on the whole approach: it assumes private delivery. If your images must be anonymously reachable at a stable URL forever, presigned GETs are the wrong shape — signature lifetimes are bounded, seven days at the outside for SigV4 — and you'd be better off with a public bucket behind a CDN and a different security model. Stick with a proxying app tier when files are small, uniformly trusted, and you genuinely need to inspect every byte in-process. For everything in between, the boundary above holds: signed key, direct parts, explicit commit, separate worker, private derivatives.

## Further reading

- [OWASP File Upload Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/File_Upload_Cheat_Sheet.html)
- [DigitalOcean Spaces documentation](https://docs.digitalocean.com/products/spaces/)
- [Amazon S3 multipart upload limits and part sizes](https://docs.aws.amazon.com/AmazonS3/latest/userguide/qfacts.html)
- [sharp API — resize and constructor options](https://sharp.pixelplumbing.com/api-resize)
- [MDN: Cross-Origin Resource Sharing (CORS)](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CORS)
