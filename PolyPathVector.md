---
Summary: Composite immutable vector — an ordered chain of child IVectors whose discretization emits multiple contours so gaps survive round-trips. The canonical multi-segment shape; rectangles, wall paths, and stroke-outline polygons all reduce to this type.
Tags: #vector #immutability #composite #map-geometry #atlasforge
---

# PolyPathVector

## Identity & shape

`PolyPathVector` is the concrete `ICompositeVector` with `type: 'polyline'`. It carries:

- `id: string` — stable identifier, used by [[AtlasForgePixiCanvas]] as the cache key.
- a private ordered `ReadonlyArray<IVector>` of children (lines, arcs, beziers, or further composites),
- a private `PathStorage` map of shortest paths between children (used during subsection edits),
- an immutable `VectorRuntimePolicy` reference (the singleton from DI; see [[Vector-Math-Extensions]]),
- a derived `_isClosed: boolean` — true when the child chain forms a loop (last child's end equals first child's start under `policy.quant.eq`).

All fields are `readonly` or behind protected accessors. The constructor is **protected**; production code may only construct via `VectorFactory.createPolyPathVector(id, vectors)` or the internal `createPolyPathVectorInternal(policy, vectors, id)` factory. The `vector-constructor-lockdown.test.ts` invariant test and the CI `scripts/guard-vectors.mjs` guard enforce this.

## Data exposed via the interface

`PolyPathVector` implements `IPolyPathVector`:

- `getVectors(): ReadonlyArray<IVector>` — ordered children.
- `withNewVector(v: IVector): IPolyPathVector` — append; returns a fresh instance.
- `withUpdatedVector(v: IVector): IPolyPathVector` — replace by `v.id`; returns a fresh instance.

Plus the base `IVector` surface (see [[Vector-Math-Extensions]] for the math behind these):

- `discretize(policy): SegmentPath` — concatenates each child's discretization. Disjoint children produce **multiple `SegmentContour`s**, never null segments. `closed` is propagated per contour.
- `getBoundingBox(policy)`, `getLength(policy)`, `getHitBoxes(policy)`, `getInteractionPoints(policy)`, `containsPoint(p, t, policy)` — all delegate to `pathBBox` / `pathLengthRounded` / `pathHitBoxes` / `pathSnapPoints` / `pathContains` from `geometry/kernel-path.ts`.
- `getSubsection(start, end, threshold, policy)` — projects the two picks onto the child chain via the `PathStorage` shortest-path map and returns an `IPolyPathSubsection`.
- `withSubsectionRemoved(sub, policy)` — yields `EmptyVector`, a trimmed `PolyPathVector`, or splits into two if the removed range cuts a closed loop.
- `withTranslation(dx, dy, policy)` — maps `withTranslation` over every child; returns a new `PolyPathVector` with the same `id`.
- `equals(other)` — structural: same type, same child count, each child `equals` in order. Used by [[AtlasForgePixiCanvas]] for cache dirty-checks.
- `equalsShape(other, policy)` — defers to the base `Vector.equalsShape` which compares normalized `SegmentPath`s via `equalsSegmentPath`.

## Immutability guarantees

1. **No in-place mutation.** Every `with…()` returns a new instance; the original is untouched.
2. **Children are immutable too.** `withUpdatedVector` and `withNewVector` rebuild the array via spread; they never push or splice the existing slot.
3. **Policy is captured once.** The singleton is bound at construction and re-used for every derived computation. Two `PolyPathVector`s constructed against the same policy share the policy by reference.
4. **Same `id` across builders.** `withTranslation`, `withNewVector`, `withUpdatedVector` all preserve `id` so caches treat them as the same logical entity for the dirty-check fast path while still failing the slow-path `equals(other)` check (which compares children).

## Interface boundaries

- **Upward** — domain code consumes only `IPolyPathVector`. Map items (`Wall`, `Tile`, `Contour`, `Shadow`, `ObjectPath`, `StairPath`) hold the abstract `IVector` and never reach for `PolyPathVector` directly.
- **Downward** — `PolyPathVector` calls into pure kernels from [[Vector-Math-Extensions]] for every analytic operation. It never touches PixiJS.
- **Sideways** — render is delegated: `renderWithOptions(canvas, options)` calls `canvas.drawPolyPath(this, options)` so [[AtlasForgePixiCanvas]] (the only renderer) decides how the shape becomes pixels.
