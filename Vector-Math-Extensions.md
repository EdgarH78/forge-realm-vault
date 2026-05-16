---
Summary: The pure-function math layer under `apps/ui/src/atlasforge/map/vectors/geometry/`. Total, deterministic, policy-parameterized kernels that compute discretization, bounds, length, hit boxes, snap points, subsection projection, stroke offsetting, and bezier fitting. No throws on hot paths; no PIXI; no mutation.
Tags: #geometry #pure-functions #determinism #kernel #atlasforge
---

# Vector-Math-Extensions

## The contract every kernel honors

Each function under `vectors/geometry/` obeys five rules:

1. **Stateless.** No module-level mutable state; nothing is memoized outside the caller.
2. **Policy-parameterized.** Quantization, equality tolerance, grid step, max-error, and segment caps all flow in via the `VectorRuntimePolicy` singleton. Two kernels called with the same policy on the same input return bit-identical output.
3. **Total.** No `throw` on hot paths. Degenerate inputs (zero-length segments, collinear control points, empty contours) return a typed `KernelResult<T>` ∈ `{ ok, value } | { empty } | { invalid, reason }` or a `RemovalResult<T>` ∈ `{ empty } | { unchanged } | { single, value } | { split, [a,b] }`. Both unions live in `packages/shared/src/vector-kernel-types.ts`.
4. **Deterministic.** Same inputs ⇒ same outputs across calls, processes, and machines. No `Date.now`, no `Math.random`, no iteration over `Set` insertion order without sorting.
5. **No allocation of domain objects.** Kernels return plain records (`SegmentPath`, `BoundingBox`, `HitBox`, `SnapPoint`). Concrete vectors are constructed only via `VectorFactory` (see [[PolyPathVector]] for an example consumer).

## The canonical discretization

`SegmentPath = { contours: ReadonlyArray<SegmentContour> }` is the single canonical discrete form. Every analytic vector (`StraightLineVector`, `ArcVector`, `QuadraticVector`, `CubicVector`, `CurvedLineVector`, `AnnulusVector`, and the composite [[PolyPathVector]]) implements `discretize(policy)` and *every* other math operation derives from that path. There are no parallel discretizations — the snap service, hit tester, brush mesher, asset tiler, and length integrator all walk the same `SegmentContour`s.

Path invariants: coordinates quantized through `policy.quant.q`; no zero-length segments; deterministic ordering; closed contours imply semantic wrap, not an explicit closing segment.

## The kernel inventory

- **`kernel-path.ts`** — the universal post-discretization layer. `pathBBox`, `pathLength`, `pathLengthRounded`, `pathContains(path, point, threshold)`, `pathSnapPoints(path, endpoints, policy)`, `pathHitBoxes(path, fallback, opts)` with default width 8 and chunk length 8. This is what every vector calls for its `IVector` surface implementations.
- **`flatten-kernel.ts`** — backed by `@flatten-js/core`; used for arc / bezier flattening and intersection.
- **`kernel-bezier.ts`** — bezier split + sample, backed by `bezier-js`.
- **`kernel-arc-by-points.ts`** — discretize an arc from `{center, rx, ry, start, end, isFullShape}`; respects `policy.discretize.minSegments` so circles never look polygonal at low zoom.
- **`bezier-fit.ts`** — least-squares fit of a smooth bezier through a polyline; consumed by `CurvedLineVector.withNewPoint`.
- **`corner-split.ts`** — corner detection at `cornerAngleDeg` (default 135°) for the stroke-fit pipeline.
- **`reflex-fan.ts`** — reflex-corner fan-out for mitered wall ribbons.
- **`snap-intersection.ts`** — segment/segment intersection for snap heuristics.
- **`stroke-outline.ts`** — centerline → closed outline at radius `r`, with semicircular end caps; emits either a smooth `CurvedLineVector` or a closed [[PolyPathVector]] (`asPolygon: true`) for stable preview rendering.
- **`stroke-fit-pipeline.ts`** — orchestrates corner-split → bezier-fit for the freehand pen.
- **`segment-path.ts`** — `equalsSegmentPath(a, b, policy)`, the workhorse behind `Vector.equalsShape`. Direction-insensitive for open paths.
- **`distance.ts`** — `distanceFromPointToSegment`, `projectPointOntoSegment`.
- **`quantize.ts`** — re-exports policy quantization helpers for kernels without direct policy access.

## How it plugs in

`VectorRuntimeOps` wraps the kernels for ergonomic call sites: `ops.discretize(v)`, `ops.bbox(v)`, `ops.length(v)`, `ops.interactionPoints(v)`, `ops.projectPoint(p, v)`, `ops.distanceToVector(p, v)`, `ops.snapToGrid(p, r)`. The snap service, drag tools, and the eraser tool all go through `ops`; [[AtlasForgePixiCanvas]] reads `ops.discretize(v)` only when it needs raw segments for tiling and glow paths.
