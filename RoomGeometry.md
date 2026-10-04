---
Summary: **Room geometry** on a building level is a **scale tier** (`s|m|l|xl`) plus an **aspect bucket** (one of Gemini's ten `aspectRatio` values) — never free width/height. `ROOM_SHAPE_TABLE` in `packages/schema/src/domain/RoomGeometry.ts` is THE table (37 legal pairs); grid dimensions, the image's aspect parameter, feet, and furniture density are all *derived* from it. Ownership is split by step: the rooms planner chooses the tier, layout chooses the ratio and position and may never change the tier, the reference image is generated at the bucket the room's *rectangle* is (not the declared ratio) and is told the room's size in feet. The envelope is a bounding box; rooms fill it and carve at most `MAX_CARVE_FRACTION` (15%) off the edges, footprint contiguous, no interior hole. Drift tests pin the PROMPTS' restatements of these numbers to the code; the schema descriptions and the design docs are not guarded — see the restatement checklist below.
Tags: #map-forge #geometry #schema #prompts #atlasforge
---

# RoomGeometry

The furnishing step is a diffusion model, and free integers are the wrong vocabulary for it: it composes to fill its frame and it takes a fixed `aspectRatio` parameter that silently beats any prose asking for proportions. So a building room's geometry is a **shape from a finite table**, and everything numeric about the room is derived from that shape. Design of record: `.ai/plans/2026-08-11-structure-type-building-envelope/ROOM-GEOMETRY.md` (read its HANDOFF §4 corrections first). Landed 2026-08 → 2026-09 in #653–#655 and #662–#666.

## The data shape
```ts
// packages/schema/src/domain/RoomGeometry.ts
type RoomShape = { size: 's'|'m'|'l'|'xl', aspect_ratio: '1:1'|'4:3'|… }        // ONLY the 37 table pairs
type IRRoomGeometry = { x: number; y: number } & RoomShape                     // packages/schema DungeonIR.ts
type PersistedRoomGeometry = IRRoomGeometry | { x; y; width; height }         // + the pre-tier arm, READ ONLY
```
- `ROOM_SHAPE_TABLE` — the 37 legal `(tier, ratio) → {w, h}` entries, frozen at load. `s` offers 7 ratios, the rest all 10. `s + 16:9` does not typecheck and `parseRoomShape` refuses it at the model boundary.
- **Rule 1 — dimensions are derived, never modelled.** `getRoomGridDimensions(shape)` / `roomGridRect(geometry)` are the only readers. Nothing stores `{w, h}`.
- **Rule 2 — key on what is measured, not what was requested.** The reference image is generated at `getClosestAspectRatio(w, h)` of the room's *rectangle*, never at the declared `aspect_ratio` (`l 4:3` is 10×7, nearer 3:2). The returned canvas is *measured* from the bytes; an unmeasurable image is a non-fatal room failure, never a square guess.
- The pre-tier `{width, height}` arm exists only so Temporal histories recorded before the table still replay. Nothing constructs it. Consumers read both arms through the accessors.

## Who owns what (the load-bearing split)
| step | chooses | may not |
|---|---|---|
| `planRooms` (planner-rooms prompt) | the **tier** per room, `size`, required | — |
| `layout` (layout prompt) | the **ratio** within the tier, and `x, y` | change the tier — a changed tier is put back with a warn; a ratio the tier doesn't offer snaps to the nearest offered (`snapAspectRatioToTier`) |
| `genReferenceImage` (top-down-room prompt) | the image's `aspectRatio` **parameter** (single author; the YAML declares none) and the **scale block** (feet, size class, furniture band, `describeRoomScale`) | invent a size for a room with no geometry |
| `validateMap` repair | `x, y` only, schema chosen **per map** (tiered vs legacy — never a union) | de-tier a room |

The planner owns the tier because it is the step doing the area arithmetic against the envelope; a floor that will not pack is then visibly a planning fault (carve), not something layout hid by resizing. Decided with Edgar 2026-09-06.

## The fill contract (one contract, three prompts)
The envelope is a **bounding box**, declared by the levels planner, never derived from the rooms — deriving it is the dungeon algorithm in a building's vocabulary. Rooms fill it from the edges in; the leftover is carved off the **edges**, at most `MAX_CARVE_FRACTION = 0.15`; the footprint is one 4-connected shape; every carved region touches the box edge (no interior hole). The planners sum tiers to 85–100% of the box. A tier's area is a **band** (`m` is 20–36 squares), so the planners count the areas they have in mind, not band maxima, and layout prefers the larger shape when the box has room, squarer on a tie.

Enforcement today: `DungeonIRValidator` **reports** contiguity and interior void (not structural); the **live harness** gates on them and on the carve ceiling, measured against the floor's *own* box, which is weaker than the declared one. Runtime enforcement of the ceiling is reachability-gate slice 5, not built.

## Scale vocabulary (`FT_PER_GRID_SQUARE = 5`)
| tier | squares | sq ft | reads as | major pieces |
|---|---|---|---|---|
| s | 6–12 | 150–300 | bedroom, pantry, small office or closet | 2–4 |
| m | 20–36 | 500–900 | kitchen, parlour, shop floor | 4–6 |
| l | 70–84 | 1,750–2,100 | taproom, hall, workshop | 8–12 |
| xl | 150–196 | 3,750–4,900 | great hall, warehouse, nave | 12–20 |

`describeRoomScale(shape)` is the single source for the prompt: feet, tier, reads-as, furniture band. "A chair is about 3 ft" is what puts the room in the model's training distribution; "units" is not.

**There is no `xs` tier** (removed 2026-09-19, Edgar's call). `s` is the floor — 6–12 squares, 150–300 sq ft — and it absorbed the closet/privy vocabulary (leading with the MIDDLE of the band: this string is the only free text the generator gets about scale, and "closet" first anchors a 300 sq ft room small). The deletion cost nothing: every rectangle `xs` offered (3x3, 3x2, 2x3) is also an `s` rectangle, so the table lost a tier and not a single shape.

How it got there: `xs 1:1` was 2x2, and the generator will not draw a 10 ft square room — a privy planned 2x2 came back drawn 8x8, quarter-scaling everything in it. The floor rose to 3x3 (2026-09-08), which also forced the `xs` furniture band 1–3 → 2–4 since the band is AREA-derived and 1–3 was priced for a 100 sq ft room. At that point `xs` was a strict subset of `s` in shape menu, area band AND furniture band, and only the phrase "closet, privy or stair landing" still distinguished them. This supersedes HANDOFF §3.2's "2x2 is correct"; that decision's surviving half is the aspect bound replacing the short-side bound.

**Replay — `RETIRED_TIER_ALIASES`.** A history recorded before the removal carries `size: 'xs'`, and the type says it cannot. `resolvePersistedTier(size)` maps a retired tier to the live one that replaced it (`xs → s`) and is consulted by `getRoomGridDimensions`, `tierAreaRange` and `describeRoomScale`. It is for PERSISTED data only — `parseRoomShape` is the model boundary and must keep refusing a retired tier, or the prompt's menu and the table quietly disagree. `layout.ts`'s `buildingBrief` also resolves the PLANNED tier this way, and REFUSES one it cannot — a level that finished `planRooms` before the deploy and reaches layout after would otherwise die at `parseRoomShape` with a level's paid work already done. It resolves there rather than at `tieredGeometry` because `buildingBrief` runs before `prompt.render`: the refusal then costs nothing on each of layout's three retries, and the model is shown a tier its own response enum offers.

An alias is only honest when it is LOSSLESS, and `xs → s` is: every rectangle `xs` offered is the `s` rectangle at the same ratio, so a replayed room keeps its size. Without it the unknown-tier path drops to the table's SMALLEST rectangle — a 225 sq ft room becomes 150, and the paid `imageConfig.aspectRatio` flips from `1:1` to `3:2`. Pinned by a test that compares every retired pair against its `s` twin. A future retirement that cannot be expressed this way needs a nonRetryable replay guard in `levelWorkflow`'s own `buildingBrief` (which has two already, for plans predating room ids and room tiers), not an alias entry.

**Losslessness is the precondition for everything downstream.** It is why a retired tier reaching a mid-flight `validateMap` repair payload can be ignored — the enum there excludes it, and the model either echoes the live tier (a silently correct retiering) or burns one repair attempt. A LOSSY retirement breaks that, and would need normalisation where the IR is read for the repair call as well as the `buildingBrief` guard. Check this before adding the next alias, not after.

**Where this table is restated**, so a change is a checklist rather than a recall: the two planners' table rows and `map-forge-layout`'s shape-menu line (drift-tested); layout's rule-5 prose example (NOT tested — it names a rectangle); `map-forge-planner-rooms/schema.json`'s `size` description (NOT tested); `plannerLevelsPromptLoad.test.ts`'s hardcoded ranges; `map-forge-top-down-room`'s `room_size_tier` variable description (NOT tested — the live reference prompt); the corridor-gate rationale in `buildingAssertions.ts`, its test, `mapForgeLive.integration.test.ts` and `workedExample.test.ts` (all four explain the gate in terms of which shapes are 2 squares narrow); `ROOM-GEOMETRY.md`'s tier table, worked examples, ladder-span figure, fidelity table, lumpy-ladder list, the "xs 1:1/4:5/5:4" line, the whole `xs` row and the `TIER_RATIO_DIMENSIONS` code block, all corrected via `HANDOFF.md` §4 rather than in the body; this atom. Only the drift-tested items are caught automatically.

**A cross-tier collision is NOT harmless the way a within-tier one is.** Two tiers sharing a rectangle share a bucket, but `describeRoomScale` keys `readsAs` and `furnishing` on the DECLARED tier and both reach the reference prompt — one room, two descriptions. Rule 2 covers the bucket only. That is what `xs` had become, and why it is gone.

## Drift tests — prose restating a code constant must be pinned
- `tierVocabularyDrift.test.ts` — the planners' area/sq-ft table rows == `ROOM_SHAPE_TABLE`, and the set of restating prompts is asserted.
- `layoutShapeMenuDrift.test.ts` — layout's per-tier shape menu == the table entry for entry, both ways; both `schema.json` enums == the constants.
- `fillContractDrift.test.ts` — all three planning prompts state `at most 15%` from `MAX_CARVE_FRACTION`, call the envelope a bounding box, carry none of the exact-tiling sentences, and each carries its half of the band guidance.
- `workedExample.test.ts` — the planners' worked example is executable calibration: it must tile, sum, and be reachable.
A pin must pin the **sentence**, not a word (a bare `/band/` probe survived deleting the guidance), and must be **mutation-checked** before it counts. Prose-pinning can only blacklist what has been written before.

## Replay
Adding a required field to the rooms plan (`id` in #653, `size` in #662) is guarded at `buildingBrief` (workflow code, re-executed on replay) with a non-retryable `ApplicationFailure`; never by relaxing the field to optional. The orchestrator is not deployed to prod, which is what makes an un-`patched()` guard acceptable today.

See [[MapForgeAgent]] for where these steps sit in the workflow tree, [[Type-Discipline]] for why the pre-tier arm is a separate type rather than an optional, [[Architecture-Decoupling]] §7 for prompt externalisation (a per-invocation generation *parameter* derived from domain data is authored in code, not the YAML).
