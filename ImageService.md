---
Summary: The .NET 8 stateless image-processing microservice on port 5100. Phase 8 D-02 made it processing-only: `POST /api/process` runs the chroma-key background-removal pipeline (+ deterministic bbox-crop), denoise, resize, and WebP encoding. SAM (AI-detection) removal was deleted 2026-08-03 — removal is chroma-key only, with no default (unresolvable requests → HTTP 422). JWT-validated with the same HS256 secret as [[API-Service]], [[Worker-Service]], and [[RenderService]]. CPU-bound; no database; same input → same output (pure pixel ops, no model inference).
Tags: #service #image-service #dotnet #processing #atlasforge
---

# ImageService

## Stack & shape

`apps/image-service/` is a .NET 8 ASP.NET Core app — `mcr.microsoft.com/dotnet/aspnet:8.0` base image, `Dockerfile.image-service` build. Stack:

- **SixLabors.ImageSharp** + `ImageSharp.Formats.Webp` — pixel ops + encoding.
- **Microsoft.AspNetCore.Authentication.JwtBearer** — HS256 token validation.

No ML model is loaded (the SAM/ONNX model-builder Docker stage and MobileSAM/SAM-2 checkpoints were deleted 2026-08-03). Port 5100. `/health` returns 200 immediately — pure CPU pixel service, no model warm-up; container healthcheck `curl -f http://localhost:5100/health` gates the [[Worker-Service]]'s `depends_on`.

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
| `chroma-key` | **Alias only.** Resolves to `chroma-key-flood`; the standalone pixel sweep is deleted. It removed every matching pixel with no connectivity check, which is how it ate asset detail. Accepted so persisted attempts naming it still process. Magenta `#FF00FF` is the production default (zero hue overlap with stone/iron/wood); see [[AgenticImageGenerationPipeline]] §generate. |
| `chroma-key-flood` | **Discover-then-flood.** Scans the *whole* image for seeds: a pixel seeds a BFS only inside a tight hue window (`SeedHueTolerance` 8 deg), and the flood then expands through the wide window (25 deg) to catch the anti-aliased fringe. The chroma processors auto-detect the background from the image corners and key on the colour **actually measured**, falling back to the supplied `backgroundTargetColor` when detection is inconclusive. The palette snap is *identity only* — it names the chroma family, it is never the key. Keying on the canonical palette hue was a defect (260815): Gemini renders magenta 15-35 deg off 300, so the tight seed window centred on 300 admitted nothing and a uniform, trivially keyable background was reported as a total removal failure. A corner is admitted when it is chromatic AND either snaps tightly (10 deg) to a palette entry or sits within 45 deg of the **assigned** colour; the largest agreeing group of **at least two** corners wins (a lone corner is never a background), ties broken by proximity to the assigned colour — and when the assignment is achromatic there is no anchor, so a tie is *refused* rather than resolved by scan order. Finally the candidate is rejected if it also fills the **centre** patch, within a fixed 25 deg `CentreGuardHueWindow` — wider than corner agreement (10 deg), because the flood reaches a centre that never seeds by expanding out of the adjoining background. That window is deliberately NOT the request's `backgroundTolerance`: that field is a per-caller recall knob (production sends 20 on generate, 45 on reprocess), and at 45 it would equal the assigned-hue anchor, refusing every anchored candidate with any chromatic centre — on reprocess, where no refiner sits behind the residual gate to retry. A safety bound must not move when someone tunes recall. Where a caller sends a tolerance above 25 deg, a centre in the gap can still be flooded; that band is accepted rather than blanket-refusing reprocess. Hue alone cannot tell a background from an asset: under a yellow assignment every timber tone (oak 21 deg from yellow, brass 17 deg, wood 32 deg) sits inside the anchor, and the refiner switches background colour precisely on chroma-leak, the failure wooden assets are most prone to. "Is this colour confined to the border" is independent of hue, so it can refuse what hue cannot. **Known gap:** the centre guard only fires when the asset *also* holds the centre, so timber reaching two or more corners around a distinct centre (a frame around a canvas, a shelf with items) is still admitted. Detection probably cannot close that — a yellow background drifting ~11 deg toward orange lands within 10 deg of oak, where the flood's own window cannot separate them either. The durable fix is upstream: forbid the refiner a chroma that collides with the asset's measured hue. Every refusal names its reason in the log, because refusals cost a generation and are the only evidence for tuning these bounds. Getting any of this wrong floods the asset instead of the background, which nothing downstream detects (see the over-removal note below). |
| `bbox-crop` | Crop to the provided bbox without removal; trivial pass-through after framing. Uses the `boundingBox` field (supplied by [[Worker-Service]]'s `GeminiVisionBoundingBoxClient`); the only remaining consumer of that client. |

> **Seeding changed from four-corner to whole-image (this reverses 260502-iyj).**
> Corner seeding could only reach background connected to the frame, so an
> enclosed pocket — chroma through lantern glass, between chainmail links,
> behind a torch bracket — survived as a leak. Seeds are now found by scanning.
>
> The old rationale was that interior bg-matching pixels must be preserved
> ("magenta lampshade gets eaten"). That protection is now the **tight seed
> window** plus the refiner's `BG_COLOR` retry
> (`packages/asset-forge-core/src/backgroundColor.ts`), not connectivity — the
> two cases are indistinguishable by connectivity, and the keyed colour is
> chosen precisely so assets do not contain it.
>
> Consequence: the two chroma methods have largely converged. `chroma-key-flood`
> now differs from `chroma-key` only for pixels in the 8–25 deg fringe band that
> touch no tight seed.
>
> Gating asymmetry worth knowing: chroma-leak (what this fixes) is caught
> deterministically by `ResidualRatio` -> `isHardReject` (`RESIDUAL_FAIL_WEIGHT
> = 100`). Over-removal (what this permits) has **no** deterministic metric —
> `completeness` is advisory-only and does not gate retry.

> **There is now exactly ONE background-removal algorithm.** Scan the whole image
> for pixels inside a tight hue window around the keyed colour; each match seeds a
> 4-connectivity flood that expands through a wider window. Nothing else.
>
> Deleted with it: the standalone pixel-scan processor, and both of the flood's
> escapes into an RGB tolerance scan (the achromatic-target branch and the
> low-removal-ratio fallback). Those were the pixel scan under another name.
>
> A background that cannot be keyed is now a **failure, not a degraded success**:
> nothing is removed and `ResidualRatio` reports 1, so `isHardReject` fires and the
> refiner retries the background COLOUR. `MinChromaKeyRemovalRatio` changed meaning
> — it was the fallback trigger, it is now the failure threshold.

There is **no default method**. `removeBackground:true` with no resolvable method (Decals falls back to `chroma-key`), or a chroma-key method with no `backgroundTargetColor`, throws `PostProcessingValidationException` → **HTTP 422** (`PostProcessingPipelineFactory` + `ImageProcessingController`). This replaced the old silent `else → sam` default when SAM was removed.

Post-removal stages (apply to all methods):

1. **Denoise** — small-radius blur to clean alpha edges.
2. **Resize** — if `downscaleResolution` is set, resize the longest edge to that value preserving aspect (used for the `DOWNSCALE_CATEGORIES` from [[AgenticImageGenerationPipeline]]: walls, objects, doors, windows, tileableobjects, stairs).
3. **WebP encode** at quality 90 — returned as `image/webp` binary response.

## POST /api/generate — legacy

`ImageGenerationController` routes on a `Provider` field to one of `OpenAiImageGenerator`, `ReplicateImageGenerator`, `GeminiImageGenerator`, or `StubImageGenerationService`. Phase 8 (260430-lx3) moved active generation to [[Worker-Service]]'s `DirectGeminiImageGenerator`, which calls Gemini 2.5 Flash directly. This endpoint is kept for backwards compatibility with the older asset-forge flow but is **not on the AgenticPipeline hot path**.

## Statelessness

No database connection. No on-disk cache that survives restart. Identical input → identical output — the pipeline is pure pixel ops with no model inference (deterministic since SAM's ONNX path was removed).

This is what makes the service trivial to scale. Cloud Run sets concurrency based on CPU; horizontal scaling is unrestricted because there's no shared state. Cold starts are fast — there is no model to load; a warm instance handles a `/api/process` call in under a second for typical 1024px inputs.

## Callers

- **[[Worker-Service]]** — `ImageServiceProcessingClient` calls `/api/process` after every gen attempt in [[AgenticImageGenerationPipeline]], and on the asset-reprocess path (`AssetReprocessHandler`). Routes per asset category: chroma-key/chroma-key-flood with the chroma bg color (Objects, Decals, etc.); bbox-crop for Walls/Doors, paired with the `GeminiVisionBoundingBoxClient` bbox. Every removeBackground call must carry a chroma method + color (or bbox-crop) — there is no default.
- **[[API-Service]]** — same client, used during the synchronous `POST /api/assets/:id/reprocess` route to regenerate previews/thumbs without going through Pub/Sub.

The service is never called from the [[UI-Service]] directly. Browser → asset bytes → signed URL → MinIO/GCS; processing is server-side only.

## What this service is NOT

It is **not** the only image-processing surface — [[Worker-Service]] also runs a Node `sharp` pipeline inside `AssetIngestionService` for generating thumb/preview WebPs from uploaded files (no BG removal needed for user uploads). The split: **AI-generated** images go through the .NET image-service for chroma-key removal; **user-uploaded** images go through the worker's sharp for resize-only.
