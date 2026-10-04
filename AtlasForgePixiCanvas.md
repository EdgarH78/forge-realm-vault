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

## Caching has two tiers, and they compare different things

This distinction is load-bearing and easy to get wrong — a whole class of "why won't my draw call run?" bugs lives here.

**Inner tier — the submission.** `cacheManager.submit({ id, kind, vector, options, execute })` decides whether to re-run one draw call. Keyed on `id` + `vector` (reference first, then `.equals()`) + **`options` equality** + asset reference. The canvas's own parameter shadows (`_assetFillShadow` et al.) implement this. Fine-grained.

**Outer tier — the layer bake.** Before any of that, the cache manager decides whether the layer needs re-drawing *at all*. Its unit is `CacheCheckItem = { id, vector, asset }` — **no `options`, no appearance/selection state.** Validity is `bakedItem.itemRef.equals(item)` across the layer's non-transient items; if nothing changed, the baked texture is reused and **the miss functions never run**, so no submission is ever reached.

The consequence that bites: anything that changes only *how* an item draws — hover, selection, any per-frame decoration passed as a `render()` argument rather than stored on the item — is **invisible to the outer tier**. `item.equals(item)` stays true, the layer stays cached, and the extra draw call silently never executes. This is why hover decoration historically routed through the unmasked `_overlayTarget`: the overlay sidesteps the cache entirely.

Two escape hatches exist for content that must draw every frame:

- **`MapItemPersistence.Transient` items** are excluded from bake validity and drawn live. Note `ShadowCacheManager.endLayer` appends live draws *after* the flattened sprite, so they composite **above** the baked texture. This is what makes pen previews and object placement previews work. But transient carries persistence semantics ([[Map]]`.filterTransientItems` strips it on save/export) and policy routing sends such items to the `Misc` layer, so it is the wrong tool for in-layer decoration.
- **Ephemeral submissions** — always executed live, never baked. `RenderSubmission.ephemeral` marks them, and `ShadowCacheManager.submit` routes them past the flatten-all bake decision, which would otherwise only run its deferred miss functions when the bake is *invalid* — silently dropping the draw on every frame a cached layer stays valid. This is how in-layer decoration is drawn without teaching either the caller or the cache about the other: callers stay cache-unaware and just issue a fill.

  Two rules govern where an ephemeral draw lands. **Across layers** it obeys the normal stack, which is what lets a floor-tile hover highlight render *beneath* the objects resting on it (`FLOOR` 3 → `BASE_SHADOWS` 4 → `OBJECTS` 5). **Within** its own layer it always sorts **last**, after every non-ephemeral submission, preserving `drawOrder` within each group. Without that rule the same highlight rendered below an overlapping sibling while a bake was invalid and above it once baked — the z-order flipped on a ~200 ms debounce. Consequence to keep in mind: a decoration on an item in a layer whose members *do* overlap (e.g. `OBJECTS`) draws above all of its siblings, including ones visually in front of it.

  **The frame-clear contract, and the test trap it sets.** Every frame clears the display tree before rebuilding it: `FlattenedCacheManagerImpl.renderStart` calls `removeChildren()` on *every* layer container, and `AtlasForgePixiCanvas.beginFrame` does the same for the overlay target. Nothing can therefore be "stranded" from a previous frame by construction. The consequence for tests: an assertion that a decoration is *absent* after clearing hover is a **canary, not eviction coverage** — it merely re-pins "this frame submitted no decoration", and it would pass identically against an implementation with no eviction logic at all. Worse, the one stranding that is plausible — a hover tint captured into the flattened bake — removes the live display object too, so such an assertion passes while the tint persists on screen. This has been mis-stated three times in one test file's comments; state what the assertion actually pins.

  **Precondition, currently enforced only by documentation:** an ephemeral draw must be conditional on transient view state. It is safe today because the bake builds a separate canvas and re-renders every item with `SelectionState.None`, so a selection-gated draw is simply absent at bake time. An *unconditional* ephemeral draw would be issued during the bake through a canvas whose cache manager ignores the flag — baked into the texture *and* drawn live above it, doubling its alpha permanently.

Beyond those, there is no other dirty signal. Touching a vector in place would silently poison the cache by claiming "same" when the shape changed; this is why [[Vector-Math-Extensions]] kernels never mutate input and [[PolyPathVector]] builders always return new instances.

**Region tiling is currently dormant.** `FlattenedCacheManagerImpl` implements per-region 2048×2048 `RenderTexture` tiles keyed `layerIdx:tileX:tileY`, but that path is driven by `check()`, whose only caller delegates downward from `ShadowCacheManager` — nothing above ever calls it. So `_pendingChecks` stays empty, `_endItemTrackingLayer` is inert, and every layer takes the flatten-all path against a single pseudo-region covering the whole layer. Invalidation therefore re-bakes an entire layer, not a tile. Treat the region machinery as built-but-unwired until a caller appears.

## Extension boundary

To add a new render mode: extend `AtlasCanvas` with the new method, implement it here, route it through `_cacheManager.submit` for free caching, and stop. Do not import `pixi.js` from anywhere else.

If the new mode is *decoration* rather than content — it depends on transient view state like hover or selection, and must redraw every frame — make it an ephemeral submission (above) instead of a plain one. Do not push the problem outward: callers must never spawn transient items, thread cache keys, or reason about bake scheduling to make a draw call land. The canvas decides; the caller just draws.
