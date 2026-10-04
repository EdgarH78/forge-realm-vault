# Readout Rotation Convention

**One convention, three places.** The placement readout's `rotation` field, the compiler's facing table, and the wall snap all had to agree on what a rotation *means*. Two of the three silently disagreed on the left/right sign until 2026-09-06; this atom is the single place the sign is derived from.

## The convention (owner: the readout prompt)

Source of truth: `packages/prompt-store/prompts/map-forge-placement-readout`, "Rotation convention" — named without a version so a bump cannot strand this reference (the prompt was at v3 when this was written and is at v5 now; the convention text is unchanged). See [[Prompt-Versioning]].

- `rotation = 0` — the asset in its **natural top-down orientation**: back at the TOP of its box, front facing the BOTTOM. A chair's seat faces down; a bed's headboard is at the top; a fireplace opening faces down.
- Rotation increases **clockwise** (screen coordinates, y down). Values are one of `0 | 90 | 180 | 270`.
- `90` → what was at the top is now on the RIGHT (front faces LEFT). `180` → front faces UP. `270` → top is now on the LEFT (front faces RIGHT).
- Rotationally symmetric items (round tables, barrels, bowls) use `0`.

Front vector per rotation, in y-down world coordinates:

| rotation | front points | back rests on |
|---|---|---|
| 0 | +y (down) | top wall |
| 90 | −x (left) | right wall |
| 180 | −y (up) | bottom wall |
| 270 | +x (right) | left wall |

The generated assets honour the `rotation = 0` half of this: every Gemini object asset with a front is drawn back-at-top / front-at-bottom (verified across a tavern's chairs, benches, beds, cabinets, fireplaces on 2026-09-06). The `image-gen/objects` prompt does not say so explicitly; it is the model's default and is relied upon.

## Who consumes it

- **`compileRoom.reconcileRotation`** (`apps/orchestrator/src/activities/compileRoom.ts`) — the `FRONT` table is this table. The readout never sees the asset, so its `rotation` is a *guess* about the natural orientation, while its `box_2d` is *measured* off the reference image. The box therefore decides whether the sprite is upright (`{0,180}`) or on its side (`{90,270}`); the readout rotation is kept when it agrees and otherwise replaced by the agreeing turn whose front faces the cluster anchor (else the room centre). Squarish assets or boxes (aspect 0.8–1.2) carry no orientation signal and pass the readout rotation through — this is what preserves D-15 ("trust the LLM rotation") for chairs. The sprite box is the readout box turned a quarter when the asset is on its side.
- **`DungeonIRCompiler.snapWallFurnishing`** (`packages/map-ir`) — position only. It insets the item against the nearest wall by the half-extent perpendicular to that wall, using the footprint **after** rotation (a quarter turn swaps w/h). It never sets rotation: its old "parallel to the wall" rule assumed a landscape asset and used left = +π/2 / right = −π/2, which under this table puts the FRONT into the wall.
- **The renderer** applies `rotation` as PIXI's clockwise-positive radians. Compiled maps store radians; the readout emits degrees; `compileRoom` converts once.

## Rules

1. Any code that maps a wall side or a target direction to a rotation derives it from the table above, never from a local sketch.
2. The readout box is authoritative for orientation; the readout rotation is authoritative only for facing, and only when it is consistent with the box.
3. Changing the prompt's convention means changing `FRONT` and this atom in the same PR.

Related: [[MapForgeAgent]], [[RoomGeometry]], [[Architecture-Determinism]].
