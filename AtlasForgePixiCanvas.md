---
Summary: The single PixiJS adapter. A one-way projection layer that consumes immutable IVector and IMapItem state and writes PixiJS display objects. The only file in the subsystem allowed to import `pixi.js`. PIXI version bumps must not cascade past this boundary.
Tags: #renderer #projection #pixi #adapter #atlasforge
---

# AtlasForgePixiCanvas

## Role: dumb projection layer

`AtlasForgePixiCanvas` (`apps/ui/src/app/canvas/AtlasForgePixiCanvas.ts`) implements the `AtlasCanvas` interface — a shape-in, pixels-out surface. It computes no geometry. It receives an immutable vector (typically a [[PolyPathVector]] or one of its analytic siblings), asks the vector for its already-deterministic `discretize(policy)` output (see [[Vector-Math-Extensions]]), and translates that into a `Graphics`, `TilingSprite`, `Mesh`, or `Sprite`. The data flow is strictly one-way: domain state → PIXI display tree. Nothing observed in PIXI flows back into the vector model.

## What it owns

- A PIXI `Application`, a root `Container`, and the active `ICacheManager` (defaulting to `DirectCacheManager`; the main editor swaps in `FlattenedCacheManagerImpl` with region-tiled `RenderTexture` caches).
- The captured `VectorRuntimePolicy` (singleton, sourced via DI) — used when the canvas itself needs `vector.discretize(policy)` to walk segments for tiling or glow.
- A `ViewPort` reference for `_scale = baseScale * zoom`. This scale is *divided into* every world-space stroke width so a 1-pixel stroke at 100% remains 1 pixel at any zoom.
- A `_textureCache: Map<assetId, Texture>` of decoded base64 images, with a `_pendingTextureCount` + callback list so async decodes notify the level cache when ready.
- Parameter caches keyed by `id` + parameter fingerprint: `_assetFillShadow`, `_intensityMaskCache`, `_softnessMaskCache`, `_baseShadowRTCache`, `_steppedGradientCache`, plus a single reused `_dropShadowBlurFilter`.
- An overlay path: `_overlayTarget` and `_overlayCache` for edit handles drawn outside the masked map bounds.

## The projection rules

1. **No mutation flows back.** PIXI display objects are leaves of the tree; the canvas never reads positions off a `Graphics` to update a vector. Vectors only change through `with…()` builders called by tools above the renderer.
2. **Reference equality first.** `_submitGraphics(id, vector, options, execute)` and the cache manager both compare `vector` by reference before falling back to `vector.equals(other)` (structural). Identical references ⇒ guaranteed cache hit ⇒ no `execute()` callback. This is the load-bearing reason that [[PolyPathVector]] and every other vector are immutable: same reference *is* same shape.
3. **`AtlasCanvas` is dumb.** Every method takes already-meaningful state (an `IStraightLineVector` for `drawLine`, an `IArcVector` for `drawArc`, a closed `IVector` for `drawAssetFill`, a `CanvasImage` for textures). The canvas does no snapping, no hit testing, no subsection cutting, no map-item-kind branching.
4. **Transforms are rendering-only.** When `VectorDrawOptions.transform` carries a 2D affine `Matrix` (object rotation, asset rotation, mirror), `_applyTransform` decomposes it into a PIXI `pivot` + `position` + `rotation`. The underlying vector's analytic identity is unchanged — it is the *display* that rotates.
5. **The viewport, not the canvas, owns world↔screen.** The `ViewPort` provides `worldToScreen` / `screenToWorld`; the canvas only consumes `_scale` to compensate stroke widths. Pan and zoom updates publish through `viewPort.onChanged`, which triggers `requestRender`.

## The submission protocol

`Map.render(canvas, _, cacheManager, levelCacheManager)` brackets the frame:

1. `cacheManager.renderStart()` runs LRU eviction and clears per-frame display children.
2. For each visible layer: `beginLayer()` → map items call canvas methods, which call `cacheManager.submit({ id, kind, vector, options, execute })` → `endLayer()` resolves hits vs. misses for this layer's regions.
3. `commitFrame()` adds display objects back to the parent container.
4. `levelCacheManager.flush()` runs *after* commit so below-level previews survive the per-frame child clear.

Every cache decision is a function of `id` (vector id) + `vector.equals(other)` + options equality. There is no other dirty signal. Touching a vector in place would silently poison the cache by claiming "same" when the shape changed; this is why [[Vector-Math-Extensions]] kernels never mutate input and [[PolyPathVector]] builders always return new instances.

## Extension boundary

To add a new render mode: extend `AtlasCanvas` with the new method, implement it here, route it through `_cacheManager.submit` for free caching, and stop. Do not import `pixi.js` from anywhere else.
