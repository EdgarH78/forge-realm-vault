# Architecture: Determinism & Visual State

We handle a lot of geometric logic in AtlasForge, and if state mutation wanders across the codebase, the canvas drifts, the visual sync shatters, and debugging becomes an absolute nightmare. We avoid this by enforcing strict, interface-driven immutability.

## The Core Strategy
The visual rendering engine (PixiJS) is completely dumb. It is a pure, one-way projection of a single source of truth. It does not calculate geometry; it does not track local mutations. It reads an immutable interface and paints pixels.

### 1. Immutability is Non-Negotiable
Every geometric operation (translations, scaling, vector math) must return a *brand-new instances* of the visual state. We do not modify properties on an existing vector object. 

```typescript
// ❌ WRONG: Destructive, unpredictable in-place mutation
function offsetPath(path: PolyPath, dx: number, dy: number) {
    path.points.forEach(p => { p.x += dx; p.y += dy; }); 
}

//  RIGHT: Pure, deterministic mapping. Returns a new projection.
interface IImmutableVector {
    readonly x: number;
    readonly y: number;
    add(other: IImmutableVector): IImmutableVector;
}