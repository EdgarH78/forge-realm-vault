---
Summary: The aggregate root for an authored map. An immutable record of `size`, `renderingMode`, an array of `Level`s (vertical slices), a `currentLevel`/`currentLayer` cursor, and an `ILayerPolicy` that routes new items to their semantic home. Every mutation returns a fresh `Map`. Its `render()` method is the only place the model crosses into [[AtlasForgePixiCanvas]].
Tags: #map #aggregate #immutability #atlasforge
---

# Map

## Identity & shape

`Map` (`apps/ui/src/atlasforge/map/foundation/map.ts`) implements `Renderable`. Construction takes `(size, renderingMode='basic', layers, currentLayerIndex, name?, id?, layerPolicy?, levels?, currentLevelIndex?)`. If `levels` is provided it wins; otherwise the constructor wraps `layers` in a single default `Level` (elevation 0..10). The cursor pair `(currentLevelIndex, currentLayerIndex)` is the editor's "where am I right now" pointer.

Derived getters:

- `layers` — `levels[currentLevelIndex].layers`
- `currentLayer` — `layers[currentLayerIndex]`

See [[Layer-and-Level]] for the container shapes.

## The mutation surface

Every method below returns a *new* `Map`. The originals are untouched, the same way [[PolyPathVector]] vectors evolve through builders.

- `withLayers(layers)` — replace current level's layers.
- `withItem(item)` — route through `layerPolicy.resolveLayerIndex(item)`; if the resolved layer is out-of-range, fall back to `currentLayerIndex`. Existing same-id replaces; new id appends.
- `withItemInPlace(item)` — preserve the item's current layer even if the policy says otherwise (used for wall-attached objects pinned to WALLS). Searches every layer for the id, updates on the topmost match, removes duplicates from lower layers.
- `withItems(items)` — broadcast to every layer (per-layer `withItems`).
- `withItemRemoved(itemId)` — search all layers and remove.
- `addItem(item)` — throws `DuplicateIdError` if any layer already holds that id.
- `withName`, `withId`, `withCurrentLayerIndex`, `withSize`, `withRenderingMode`, `withLevels`, `withCurrentLevelIndex` — trivial field swaps.
- `addLayer(layer)`, `removeLayer(layerName)` (clamps `currentLayerIndex` if it falls off the end), `updateLayer(layerName, updatedLayer)` (cross-layer dup-id check before swap).

## Spatial queries & bounds

`getItemsNear(layerIndex, point, radius)` is the broadphase: every item whose hitbox intersects a circle at `point`. Precision refinement is the caller's job (typically using `IVectorRuntimeOps.distanceToVector`).

`getMapBounds()` reads the `TERRAIN_BACKGROUND_TILE_ID` tile in the `TERRAIN_BACKGROUND` layer and returns `{minX,minY,maxX,maxY}` or `null`. This is the **export crop window** — a missing or undersized terrain-background tile will crop the export. The dependency is intentional but fragile (tracked as `project_export_bounds_bg_tile_dependency`).

## The render orchestration

```
render(canvas, selectionService?, cacheManager?, levelCacheManager?):
  bounds = getMapBounds()
  current = levels[currentLevelIndex]
  if current.belowLevelPreview?.enabled && levelCacheManager:
    handle = levelCacheManager.getCachedLevel(levels[currentLevelIndex+1],
              c => renderLevel(currentLevelIndex+1, c))
    if handle: levelCacheManager.drawCachedLevel(handle, intensity)
  cacheManager?.renderStart()
  for each layer in current.layers:
    cacheManager?.beginLayer()
    if layer.visible && !layer.locked:
      layer.render(canvas, SelectionState.None, selectionService, bounds)
    cacheManager?.endLayer()
```

`renderLevel(levelIndex, canvas)` is the no-selection variant used by the level cache for background bakes. Errors per layer are caught + logged; one bad layer must not crash a frame.

## Persistence boundary

`filterTransientItems()` returns a new `Map` with every `MapItemPersistence.Transient` item stripped across every level. Called once by [[MapSerializer]] before save and once before export. Anything ephemeral (preview brushes, cursor ghosts, drag overlays) dies here.

`createDefault()` produces an Untitled 1000×1000 map with a single Level and `currentLayer = WALLS` (the canonical authoring entry point).
