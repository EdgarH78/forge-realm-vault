---
Summary: Physical files live in cloud storage, deduplicated by content. `AssetFile` rows are keyed by SHA-256 hash; multiple `Asset` entities can reference the same file. Storage paths shard on the first 4 hash chars, files cap at 5 MB by CHECK constraint, and the `IStorageService` interface (see [[Architecture-Decoupling]]) swaps GCS, MinIO, and S3 implementations at the DI seam. The ingestion lifecycle (init-upload → staging → Pub/Sub → worker) decouples client uploads from the worker pipeline.
Tags: #asset #storage #files #deduplication #atlasforge
---

# AssetStorage

## AssetFile — the deduplicated row

`AssetFile` (`packages/schema/src/domain/AssetFile.ts`) is the physical-file metadata record. One row per unique file content:

```ts
interface AssetFile {
  sha256Hash: string;       // PK — 64-char hex
  gcsPath: string;          // asset_files/{first4}/{sha256}
  originalFilename: string;
  originalUrl: string;      // {gcsPath}/original/{filename}
  previewUrl: string;       // {gcsPath}/preview.webp   (1024px)
  thumbUrl: string;         // {gcsPath}/thumb.webp     (256×256)
  contentType: string;      // image/png | image/webp
  widthPx, heightPx: number;
  fileSizeBytes: number;    // CHECK <= 5_242_880 (5MB)
  firstUploadedAt, createdAt: ISO string;
}
```

The PK is the content hash, so the same image uploaded twice collapses to one row. Multiple `Asset` rows may set `assetFileSha256` to the same hash — versions of the "same" image, or two separate assets that happen to share bytes. The `Asset → AssetFile` JOIN at fetch time is `LEFT JOIN asset_files af ON a.asset_file_sha256 = af.sha256_hash`.

## Path layout

`constructAssetFilePath(sha256)` returns `asset_files/{first4}/{sha256}`. Sharding by the first 4 hex chars caps any one bucket directory at ~65k entries even at scale. `constructAssetFileUrls(sha256, filename)` returns the canonical `{ gcsPath, originalUrl, previewUrl, thumbUrl }` triple — these are the only paths the application ever generates. Image bytes outside these paths are by definition stale.

The 5 MB cap is enforced **three times**:

1. **Database** — `CHECK file_size_limit (file_size_bytes <= 5_242_880)` on `asset_files`.
2. **API route** — Fastify body-size limit on `/api/assets/upload/init` returns a signed URL whose policy enforces the cap.
3. **Storage layer** — signed URL constraints on the underlying GCS / S3 / MinIO bucket.

Defense in depth: a client that bypasses one still hits another.

## IAssetFileRepository

```ts
findBySha256(hash): Promise<AssetFile | null>
upsert(data): Promise<AssetFile>         // race-safe; idempotent
create(data): Promise<AssetFile>         // non-idempotent (will fail on conflict)
```

`upsert()` is the production path: `INSERT … ON CONFLICT (sha256_hash) DO NOTHING RETURNING *`, then a follow-up `SELECT` if zero rows came back (conflict happened — another concurrent uploader won the race). Two browsers uploading the same image from two tabs both succeed and both end up referencing the same row.

## IStorageService — the abstraction

A thin interface from `@atlasforge/schema/domain/repositories/IStorageService`:

```ts
uploadFile(path, buffer, contentType): Promise<void>
downloadFile(path): Promise<Buffer>
copyFile(srcPath, dstPath): Promise<void>
generateSignedReadUrl(path, ttlSeconds): Promise<string>
generateSignedWriteUrl(path, ttlSeconds): Promise<string>
deleteFile(path): Promise<void>
```

Implementations: `StorageServiceGCS`, `StorageServiceMinIO`, `StorageServiceS3` — all in `apps/{api,worker}/src/infrastructure/`. A factory (`createStorageService()`) reads `STORAGE_PROVIDER` from env (`gcs` | `minio` | `s3`) and constructs the right one. See [[Architecture-Decoupling]] §3 for the repository / storage decoupling rule — domain code never imports `@google-cloud/storage` or `@aws-sdk/client-s3` directly.

## The ingestion lifecycle

The path from "client has bytes" to "row in `atlasforge.assets`":

1. **Init upload**. Client `POST /api/assets/upload/init { filename, contentType }`. API validates content type (`image/png | image/webp` only), returns `{ uploadId, putUrl, stagingPath }`. The `putUrl` is a short-TTL signed PUT URL into the staging bucket.
2. **Upload bytes**. Client `PUT`s the bytes directly to the signed URL. Server stays out of the bytes path entirely.
3. **Request ingest**. Client `POST /api/assets/ingest { uploadId, stagingPath, declared metadata, userId }`. `AssetService.startIngest()` publishes one Pub/Sub message to the `atlas-asset-ingest-requests` topic and returns `{ ingestId, traceId, status: 'processing' }` synchronously. The HTTP roundtrip is now done.
4. **Worker processes**. `AssetIngestionService.processIngest(message)` (worker) runs:
   - **Download** from staging via `storageService.downloadFile(stagingPath)`.
   - **Hash** bytes → `sha256Hash`. Check `findBySha256(hash)` — if present, dedup short-circuit; if absent, continue.
   - **Validate** the declared `categoryData` against an AJV schema for the asset category. Reject if invalid.
   - **Process** with sharp: generate `preview.webp` at 1024 px, `thumb.webp` at 256×256.
   - **Move** original + derivatives from staging to the permanent path under `asset_files/{first4}/{sha256}/`.
   - **Upsert** the `asset_files` row.
   - **Create** the `atlasforge.assets` row referencing `assetFileSha256`. The `search_text` trigger fires automatically; see [[AssetFacets-and-Search]].
   - **Publish progress** events at 0.1, 0.4, 0.6, 1.0 to `atlas-asset-ingest-events`. The API subscribes and re-emits as SSE — the same Pub/Sub decoupling pattern as [[PhaseOrchestrator]].
5. **Client observes**. The SSE stream tied to `ingestId` carries `processing → thumbnails_generated → validation_complete → complete` (or `error`). The client updates its asset grid when the `complete` event arrives.

## Reprocess endpoint

`POST /api/assets/:id/reprocess` re-runs the worker's processing pipeline against the same `asset_file_sha256` — regenerates the preview/thumb. Used when sharp parameters change or a preview goes corrupt. The hash doesn't change, so the dedup row is unaffected; only the WebP derivatives are rewritten.

## Concept-hero assets

[[MapForgeAgent]]'s hero fan-out reuses this exact pipeline with two tweaks: the worker writes directly to the permanent path (no staging step — bytes come from the LLM, not the user), and the resulting `Asset` row carries a non-null `mapId` (migration 025). Everything else — the dedup, the JOIN, the signed URLs — works identically.
