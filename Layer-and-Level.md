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

From `packages/schema/src/domain/map/layers.ts` — index → name and routing role.
The indices **and** the routing rule live in `@atlasforge/schema` so producers outside the UI share them: `apps/ui/.../foundation/layers.ts` is a re-export, `DefaultLayerPolicy` delegates to `resolveLayerIndex`, and `DungeonIRCompiler` (map-ir) imports the same constants.

| Index | Constant | Role |
|---|---|---|
| 0 | `TERRAIN_BACKGROUND` | Holds the terrain-background tile whose bounds define export crop (see [[Map]].`getMapBounds`) |
| 1 | Terrain | Terrain brush strokes |
| 2 | Water | Water tiles |
| 3 | `FLOOR` | Floor tiles (closed) — **surface fill only** |
| 4 | `BASE_SHADOWS` | Base-level cast shadows |
| 5 | `OBJECTS` | `MapObject` instances — all content, whatever plane it depicts |
| 6 | `WALLS` | Walls (default editing layer in `Map.createDefault`) |
| 7 | `TOP_SHADOWS` | Above-wall shadows |
| 8 | `MISC` | Catch-all for transient editing items |
| 9 | `ROOFS` | Top-most decorative layer |

### A layer holds either surface fill or content, never both

Layer is a function of item **kind**, never of what the item depicts. Within a layer the only z-order is array index, so anything sharing `FLOOR` with the per-room floor tiles can be painted over by a tile emitted later. A rug and a wardrobe are both `kind: 'object'` and both belong on `OBJECTS`; that a rug lies flat is sort order *within* the layer, not a different layer.

`DungeonIRCompiler` violated this before 2026-08-11 by routing `elevation_role: 'floor'` furnishings to `FLOOR`. Because it emits room-by-room as `[tile, ...items]`, a later room's floor tile buried an earlier room's objects — a live crypt run silently lost a staircase and four bone piles. The compiler now treats elevation role as a plane *within* `OBJECTS` (floor → table → ceiling, bottom to top), which is how `'ceiling'` always worked.

**Sanctioned exception:** wall-attached objects (torches, brackets, door leaves) route to `WALLS`, not `OBJECTS`. `OBJECTS` (5) sits *below* `WALLS` (6), so conforming them to the kind rule would render every wall torch behind the wall image. This is the same carve-out `withItemInPlace` exists for, below.

## ILayerPolicy

`ILayerPolicy.resolveLayerIndex(item)` returns the target layer index for newly-added items. `defaultLayerPolicy` (in `DefaultLayerPolicy.ts`) narrows the live map item to its discriminators (`kind`, plus `tileType`/`shadowType`) and delegates to `resolveLayerIndex` in `@atlasforge/schema` — the shared rule, not a UI-local one. `[[Map]].withItem(item)` consults the policy; `withItemInPlace(item)` deliberately bypasses it to preserve cross-policy item placements (e.g. wall-attached objects — see the sanctioned exception above).

Producers outside the UI mirror the rule rather than calling it when they also own intra-layer ordering: `DungeonIRCompiler` imports the indices but routes by hand, because it decides the floor/table/ceiling plane order that a kind-only rule cannot express. It warns (never throws — map generation charges credits along the way) if a non-tile reaches `FLOOR`.

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
