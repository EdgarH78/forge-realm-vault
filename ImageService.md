---
Summary: The .NET 8 stateless image-processing microservice on port 5100. Phase 8 D-02 made it processing-only: `POST /api/process` runs the chroma-key / SAM background-removal pipeline, denoise, resize, and WebP encoding. JWT-validated with the same HS256 secret as [[API-Service]], [[Worker-Service]], and [[RenderService]]. CPU-bound; no database; same input → same output (modulo SAM nondeterminism).
Tags: #service #image-service #dotnet #processing #atlasforge
---

# ImageService

## Stack & shape

`apps/image-service/` is a .NET 8 ASP.NET Core app — `mcr.microsoft.com/dotnet/aspnet:8.0` base image, `Dockerfile.image-service` build. Stack:

- **SixLabors.ImageSharp** + `ImageSharp.Formats.Webp` — pixel ops + encoding.
- **Microsoft.AspNetCore.Authentication.JwtBearer** — HS256 token validation.
- **SAM model files** in `apps/image-service/models/` — MobileSAM checkpoints used by the SAM background-removal path. Loaded at startup; held in memory.

Port 5100. `/health` returns 200 once the model files are loaded; container healthcheck `curl -f http://localhost:5100/health` gates the [[Worker-Service]]'s `depends_on`.

## Auth

`Program.cs` configures `AddJwtBearer` with `ValidAlgorithms = ['HmacSha256']`, `ValidIssuer = "forgerealm-auth"`, `ValidateAudience = false`, 30-second `ClockSkew`. The single `JWT_SECRET_CURRENT` env var is shared across [[API-Service]], [[Worker-Service]], [[RenderService]], and this service — cross-service auth is just "we all trust the same HMAC secret". Every controller is `[Authorize]`.

`forgerealm-auth` is the only accepted issuer here (the `atlasforge-render` issuer is scope-bounded to [[API-Service]]'s asset-read endpoints; [[ImageService]] never sees those tokens).

## POST /api/process — the processing pipeline

`ImageProcessingController.Process(IFormFile image, [FromForm] string processingOptionsJson)`. Multipart form input:

- `image` — raw bytes (up to 50 MB; raw Gemini outputs can be large).
- `processingOptionsJson` — a JSON object specifying the pipeline. Common fields: `backgroundRemovalMethod`, `chromaKeyHex`, `tolerance`, `boundingBox?`, `downscaleResolution?`, `softness?`.

The pipeline routes on `backgroundRemovalMethod`:

| Method | Behavior |
|---|---|
| `chroma-key` | ImageSharp pixel sweep — alpha → 0 for every pixel within a `Hex + tolerance` window. Magenta `#FF00FF` is the production default (zero hue overlap with stone/iron/wood); see [[AgenticImageGenerationPipeline]] §generate. |
| `chroma-key-flood` | BFS from the four corners that respects the bg color; preserves *interior* bg-matching pixels (260502-iyj). Solves the "magenta lampshade gets eaten" failure mode. |
| `sam` | SAM model inference. Uses either a center-point prompt or a bounding-box prompt (`boundingBox` field, supplied by [[Worker-Service]]'s `GeminiVisionBoundingBoxClient`). Returns a soft alpha mask. |
| `bbox-crop` | Crop to the provided bbox without removal; trivial pass-through after framing. |

Post-removal stages (apply to all methods):

1. **Denoise** — small-radius blur to clean alpha edges.
2. **Resize** — if `downscaleResolution` is set, resize the longest edge to that value preserving aspect (used for the `DOWNSCALE_CATEGORIES` from [[AgenticImageGenerationPipeline]]: walls, objects, doors, windows, tileableobjects, stairs).
3. **WebP encode** at quality 90 — returned as `image/webp` binary response.

## POST /api/generate — legacy

`ImageGenerationController` routes on a `Provider` field to one of `OpenAiImageGenerator`, `ReplicateImageGenerator`, `GeminiImageGenerator`, or `StubImageGenerationService`. Phase 8 (260430-lx3) moved active generation to [[Worker-Service]]'s `DirectGeminiImageGenerator`, which calls Gemini 2.5 Flash directly. This endpoint is kept for backwards compatibility with the older asset-forge flow but is **not on the AgenticPipeline hot path**.

## Statelessness

No database connection. No on-disk cache that survives restart. Identical input → identical output, modulo SAM's nondeterministic mask refinement (ONNX inference can introduce tiny floating-point drift across runs but produces visually indistinguishable masks).

This is what makes the service trivial to scale. Cloud Run sets concurrency based on CPU; horizontal scaling is unrestricted because there's no shared state. Cold starts are dominated by SAM model load (~2-3 seconds); a warm instance handles a `/api/process` call in under a second for typical 1024px inputs.

## Callers

- **[[Worker-Service]]** — `ImageServiceProcessingClient` calls `/api/process` after every gen attempt in [[AgenticImageGenerationPipeline]]. Routes per asset category (chroma-key paired with magenta default; SAM paired with bbox detection).
- **[[API-Service]]** — same client, used during the synchronous `POST /api/assets/:id/reprocess` route to regenerate previews/thumbs without going through Pub/Sub.

The service is never called from the [[UI-Service]] directly. Browser → asset bytes → signed URL → MinIO/GCS; processing is server-side only.

## What this service is NOT

It is **not** the only image-processing surface — [[Worker-Service]] also runs a Node `sharp` pipeline inside `AssetIngestionService` for generating thumb/preview WebPs from uploaded files (no BG removal needed for user uploads). The split: **AI-generated** images go through the .NET image-service for SAM/chroma-key; **user-uploaded** images go through the worker's sharp for resize-only. This avoids loading the SAM model into every worker instance.
