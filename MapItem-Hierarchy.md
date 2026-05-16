---
Summary: The interface tree for everything authorable in a [[Map]]. `IMapItem` is the universal contract; `IVectorBoundMapItem` adds vector backing; `IClosedVectorBoundMapItem` enforces a closed-vector precondition at construction. Concrete kinds (Wall, Tile, Contour, Shadow, Door, Window, Prop, MapObject, ObjectPath, StairPath, SemanticItem, plus deprecated Portal) all inherit through these three rungs. Construction is gated through `MapItemFactory`; type guards are the only legal way to branch on `kind`.
Tags: #map-item #interface #immutability #atlasforge
---

# MapItem-Hierarchy

## The base contract

`IMapItem` (`foundation/interfaces/map-item-contract.ts`) defines:

- `id: string`, `kind: string` (the discriminator), `persistence: 'persisted' | 'transient'`,
- `isSelectable: boolean`, `isSnappable: boolean` (only `Wall` returns `true` — the snap heuristic is intentionally architecture-focused),
- `equals(other)` — used by cache layers in [[AtlasForgePixiCanvas]],
- `render(canvas, selectionState?)`, `withPosition(p)`, `getPosition()`, `getBounds()`, `getHitBoxes()`,
- `getHitTargets(): ReadonlyArray<HitTarget>` where `HitTarget = { item, hitBoxes }`. A `Wall` returns targets for its doors and windows *before* its own target so child clicks beat parent clicks.

## The three abstract base classes

In `foundation/map-item-base.ts`:

1. **`MapItem`** — for non-vector kinds (`Prop`, `Door`, `Window`, `Layer`, legacy `Portal`).
2. **`VectorBoundMapItem<T extends VectorBoundMapItem<T>>`** — F-bounded generic so `withVector` / `withPosition` return the concrete subtype. Owns `vector: IVector`, `policy: VectorRuntimePolicy`, `colors: CanvasColors`. Implements `getBounds`, `getPosition`, `getHitBoxes` (delegating to `vector.hitBoxes`), `containsPoint`, and `withPosition` (which converts a target position into a `vector.withTranslation(...)` delta call). Subclasses must implement `withVector(v): T`. This is what every shape-bearing kind extends.
3. **`ClosedVectorBoundMapItem<T>`** — refinement that throws if the supplied vector is not closed. The docstring explicitly acknowledges the LSP trade and the reasoning ("explicit type-system constraint beats scattered runtime checks"). Only **floor** `Tile`s are recognized as `isClosedVectorBoundMapItem`.

## The concrete-kind catalog

- **`Wall`** — `kind:'wall'`. Open or closed vector; `thickness`, `outside`, `doors[]`, `windows[]` (legacy `portals[]`), `height`, optional `image`, `inverted`, shadow params + a derived `shadow: IShadow | null`. `getRenderedVector()` returns the centerline with door/window subsections removed — that's what gets drawn as the tiled ribbon. **Only kind with `isSnappable = true`.**
- **`Tile`** — `kind:'tile'`. `tileType: 'floor' | 'terrain' | 'terrain-background'`. Floor tiles must be closed (enforced via `ClosedVectorBoundMapItem`). Optional image, intensity samples (terrain brush alpha), softness, tapers, feather.
- **`Contour`** — `kind:'contour'`. Elevation lines with an `elevation: number`.
- **`Shadow`** — `kind:'shadow'`. Vector-bound; `renderMode: fill | gradient | radial | stepped | drop`; `direction`, `intensity`, `width`, optional softness strokes, optional tapers, `rotation` (drop-shadow companions inherit their owner's rotation).
- **`Door` / `Window`** — `kind:'door'` / `'window'`. Live inside walls. Carry `segment: IVector` — a subsection of the wall centerline produced via [[Vector-Math-Extensions]]'s `getSubsection`. `Door` has `leafCount: 1 | 2`. With no image, `Door` falls back to drawing an ajar leaf rotated 25° around the hinge.
- **`MapObject`** — `kind:'object'`. **Required, immutable** texture (no `withImage` / `withoutImage`), `rotation` in radians, opacity, drop-shadow settings, derived `shadow` companion. The vector is always a closed rectangle defining the footprint.
- **`ObjectPath`** — `kind:'object-path'`. Tileable texture along a path (fences, hedges, cliffs). Offset, opacity, width, `followPath`, `inverted`, tapers.
- **`StairPath`** — `kind:'stair-path'`. Texture + perpendicular thickness; no doors, no windows, no snap, no miter joints.
- **`SemanticItem`** — `kind:'semantic'`. Vector tagged with an `Asset` reference. Map Forge attaches domain meaning ("this rectangle is a *dining table*") to a footprint without committing a texture yet.
- **`Prop`** — `kind:'prop'`. Point-only (`x, y, scale, rotation`); no vector.
- **`Layer`** — `kind:'layer'`. Container; see [[Layer-and-Level]].
- **`Portal`** — DEPRECATED. Use `Door` / `Window`.

## Type guards & construction

`isWall`, `isTile`, …, `isVectorBoundMapItem`, `isClosedVectorBoundMapItem` live alongside the interfaces and are the *only* legal way to branch on `kind`. Raw `.kind === 'wall'` comparisons widen the TS type incorrectly.

Construction is gated through `MapItemFactory` (DI key `map-item-factory`). The factory threads the singleton `VectorRuntimePolicy` and the resolved `IUIStyle.canvas` colors into every constructor, generates UUIDs when no `id` is supplied, and **builds the companion `Shadow` inline** for walls (with `shadowThickness > 0`) and map objects (with `shadows === 'on'`). The companion is a derived field — never serialized as a nested child; see [[MapSerializer]]'s Pitfall 5.
