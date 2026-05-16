---
Summary: The two container shapes that organize a [[Map]]. `Layer` is an ordered, named, visible/lockable bucket of `IMapItem`s. `Level` is a vertical slice (Ground Floor, Basement) holding a full layer stack plus an elevation range. `ILayerPolicy` routes incoming items to their semantic home — walls to WALLS, floors to FLOORS, objects to OBJECTS, etc. Below-level previews are ephemeral runtime UI state and never serialize.
Tags: #layer #level #container #immutability #atlasforge
---

# Layer-and-Level

## ILayer

`ILayer` (in `map-item-contract.ts`) extends `IMapItem` with `kind:'layer'`, `items: ReadonlyArray<IMapItem>`, `name`, `visible`, `locked`. The `Layer` class in `foundation/layer.ts` implements it.

Immutability-preserving builders (every one returns a new `Layer`):

- `withVisible(v)`, `withLocked(v)`, `withName(n)`, `withItems(items)` — last throws `DuplicateIdError` on intra-batch id collisions.
- `withReplacedItem(originalId, newItem)` — pre-checks for id collision against the rest of the layer.
- `addItem(item)` — throws `DuplicateIdError` on collision.
- `removeItem(itemId)`.
- `moveItemToFront / ToBack / Forward / Backward(itemId)` — z-order shuffles. No-ops if the item is already at the end of the move direction.

Spatial queries delegate to `IVectorRuntimeOps` (see [[Vector-Math-Extensions]]):

- `getClosestMapItem(world, kind, ops): { item, segment, distance } | null`
- `getClosestSegment(world, kind, ops): { segment, distance } | null`
- `getClosestWall(world, ops): { wall, segment, distance } | null`

`Layer.render(canvas, _, selectionService, mapBounds)` iterates `items`, culls persisted items entirely outside `mapBounds` (transient items always render), resolves each item's `SelectionState` via the selection service, and dispatches `item.render(canvas, state)`. After the primary render, it does an overlay pass over `item.getHitTargets()` so nested selectables (door / window within a wall) get their hover/select highlight drawn even when the parent's render did not handle it.

## The standard layer stack

From `foundation/layers.ts` — index → name and routing role:

| Index | Constant | Role |
|---|---|---|
| 0 | `TERRAIN_BACKGROUND` | Holds the terrain-background tile whose bounds define export crop (see [[Map]].`getMapBounds`) |
| 1 | Terrain | Terrain brush strokes |
| 2 | Water | Water tiles |
| 3 | `FLOOR_LAYER` | Floor tiles (closed) |
| 4 | `BASE_SHADOWS_LAYER` | Base-level cast shadows |
| 5 | `OBJECTS_LAYER` | `MapObject` instances |
| 6 | `WALLS` | Walls (default editing layer in `Map.createDefault`) |
| 7 | Top Shadows | Above-wall shadows |
| 8 | Misc | Catch-all for transient editing items |
| 9 | Roofs | Top-most decorative layer |

## ILayerPolicy

`ILayerPolicy.resolveLayerIndex(item)` returns the target layer index for newly-added items. `defaultLayerPolicy` (in `DefaultLayerPolicy.ts`) knows the `kind → layer` routing. `[[Map]].withItem(item)` consults it; `withItemInPlace(item)` deliberately bypasses it to preserve cross-policy item placements (e.g. wall-attached objects).

Custom policies are pluggable through the `Map` constructor's `layerPolicy` parameter, but the default is what production uses and what every test fixture assumes.

## ILevel

`ILevel` is a vertical slice. Fields:

- `id: string`
- `bottomElevation: number`, `topElevation: number` (in world units)
- `layers: ReadonlyArray<ILayer>` — a full layer stack per level
- `alias?: string` — display label ("Ground Floor", "Basement")
- `belowLevelPreview?: { enabled, intensity }` — **ephemeral, runtime-only, never serialized**
- derived `height = topElevation - bottomElevation`

`Level` class (`foundation/level.ts`) exposes `withBottomElevation`, `withTopElevation`, `withLayers`, `withAlias`, `withBelowLevelPreview` — all immutable. `Level.createDefault()` returns bottom=0, top=10, with a single default-stack `Layer`.

## Indexing convention

The `levels` array is **topmost-first**: `levels[0]` is the highest level, `currentLevelIndex + 1` is the level directly below. `[[Map]].render()` uses this to ask `ILevelCacheManager` for the below-level preview image when `belowLevelPreview.enabled` is true. The preview render is layered *under* the current level at reduced alpha so the user can trace stairs / vertical alignments. Reset on load — the deserializer never restores this field.

## Round-trip

Layer + Level state participates in [[MapSerializer]]: each `Level` writes its layers, each `Layer` writes its items via the per-kind item-serializer registry. Transient items are stripped via `[[Map]].filterTransientItems()` first, then layer visibility / locked state is persisted as-is.
