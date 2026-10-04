---
Summary: The load-bearing ordering contract for the `agent_runtime` audit tree. The review projector renders a stage's children by **merging its child-stages AND its own actions into one list and sorting by `sequence`** (`projectStageNode` in `auditProjection.ts`; stable sort, child-stages inserted first so they win ties). Therefore a stage's `sequence` is ONE shared ordering space across stages *and* actions of that parent — the direct children of any stage MUST draw their `sequence` from a single monotonic counter, never from independent 0-based spaces, or the tree silently interleaves. Corollary invariant (from `RunAuditRecorderPostgres.recordAction`): a stage is sequenced by ONE strategy — every action threads an explicit counter, OR every action uses the `MAX(sequence)+1` fallback — never both, unless the explicit sequences are deliberately placed above the fallback's range. Producers (the Temporal orchestrator workflows) own this coordination because the recorder mechanisms are per-call and blind to siblings. Governs [[MapForgeAgent]] / [[AssetForgeAgent]] audit trees; read by [[Review-Service]] / [[Review-NodeTree-UI]].
Tags: #review #tooling #atlasforge #agent-pipeline #foundational
---

# AgentRunAudit Sequencing

The `agent_runtime` audit tree ([[Review-Capture-Pipeline]]) has no explicit "order" column beyond `sequence`. The review projector reconstructs each node's child order purely by sorting on it. That makes `sequence` a **contract between producers (the orchestrator workflows that write the tree) and the projector**, and it is subtle enough that two independent reviews have had to re-derive it. This atom is that contract in one place.

## The projector merges stages + actions into one sorted list

`projectStageNode` (`apps/review-service/backend/src/services/auditProjection.ts`) builds a node's children like this:

```
const childStages = kids.get(stage.stage_id) ?? [];   // sub-stages
const ownActions  = actionsByStage.get(stage.stage_id) ?? [];  // leaf actions
const merged = [];
for (const child of childStages)  merged.push({ sequence: child.sequence, node: … });
for (const action of ownActions)  merged.push({ sequence: action.sequence, node: … });
merged.sort((a, b) => a.sequence - b.sequence);   // STABLE (child-stages win ties)
```

Two consequences that drive everything below:

1. **`sequence` is one shared space per parent.** A child-stage at `sequence: 1` and an action at `sequence: 1` on the *same* parent collide. The stable sort keeps child-stages (pushed first) ahead of actions on a tie, but that tie is a bug smell, not a design — it means two things claimed slot 1.
2. **Order is `sequence`-only.** There is no `created_at` tiebreak in the child ordering, so wall-clock write order does NOT save you. Concurrent persists that share a sequence are ordered arbitrarily (this is why reject→accept grouping is resolved via `asset_relationships`, not sequence — see [[Review-Capture-Pipeline]]).

## The invariant

> **The direct children of any stage — its sub-stages and its own actions together — must draw `sequence` from a single monotonic counter.**

Equivalently, and easier to check at a glance: **a stage should hold EITHER actions OR child-stages, not both** — because the two are written by different mechanisms with independent 0-based counters, so mixing them collides unless the producer deliberately coordinates the ranges. When a stage genuinely needs both (e.g. a loop stage with a pre-loop action, per-iteration sub-stages, and a post-loop action), the producer MUST hand out non-overlapping sequence ranges by hand.

## The recorder mechanisms (why producers must coordinate)

`RunAuditRecorderPostgres` allocates sequence three different ways, and **none of them sees its siblings across the stage/action boundary**:

| Writer | Sequence source | Scope |
|--------|-----------------|-------|
| `startStage(input)` | `input.sequence ?? 0` | explicit only — no fallback; omitting it means 0 |
| `recordAction(input)` | `COALESCE(input.sequence, MAX(sequence)+1)` | the `MAX+1` fallback is over **actions `WHERE stage_id`** — it does NOT count child stages |
| `findRootStage(...)` → `nextSequence` | `MAX(sequence)+1` over **both** child stages AND actions of the root | the resume counter, the only place that spans both tables |

So `recordAction`'s convenient `MAX+1` fallback lands the *first* action on a stage at `0` — the same slot `startStage`'s default hands the *first* child stage. That is the collision. `nextSequence` on `RootStageHandle` is the pattern that does it right (spans both), but it is only exposed for the root; sub-stages must be coordinated by the workflow itself.

The recorder's own doc comment states the narrower half of the rule: *"a given stage is sequenced by ONE strategy — either every action threads an explicit counter, or every action uses this `MAX+1` fallback. Mixing yields undefined ordering."*

## Worked example — the asset-forge generation stage

`assetForgeWorkflow` ([[AssetForgeAgent]]) is the canonical "one stage, both kinds of child" case. Its `generation` stage holds, in intended visual order:

```
generation
├─ assess            (llm-call action)        pre-loop, once
├─ attempt-1         (sub-stage)  ┐
├─ attempt-2         (sub-stage)  ├ the retry loop
├─ attempt-N         (sub-stage)  ┘
├─ persist-accepted  (asset-persist action)   ┐ post-loop
└─ persist-rejected… (asset-persist actions)  ┘
```

Three independent writers, coordinated off one workflow-local counter `genChildSeq`:

- **assess** — recorded via the `RecordingAgentRunner` decorator with the `MAX+1` fallback. It is the *only* action on the generation stage before the loop, so it deterministically takes `sequence 0`. The counter reserves slot 0 for it (`genChildSeq` starts at 1).
- **attempts** — each opens with explicit `sequence: genChildSeq++` → `1 … N`.
- **persists** — `persistAsset` receives `auditSequenceBase: genChildSeq` (= `N+1`) and threads it into `RecorderAssetAuditSink`'s `startSequence` ctor arg, so its asset-persist actions run `N+1, N+2, …`.

Result: `0, 1..N, N+1..` — three disjoint ranges, strictly ascending, no ties. **Load-bearing fragility:** this is correct *only because* assess is the sole `MAX+1` action on the stage. Add a second pre-loop `recordAction` directly on the generation stage after attempts exist and its `MAX+1` recomputes to `N+1`, silently colliding with the first persist. If a second such action is ever needed, derive the attempt base from the actual pre-loop action count (or promote the shared-counter to a `nextSequence`-style handle) rather than the literal `1`.

## The other coordination pattern — keep a stage single-kind

The map-forge parent ([[MapForgeAgent]]) avoids the mixing entirely where it can:

- The **root**'s children (planning stage, level stages, validation stage) are all sub-stages, sequenced off a manual `auditChildSeq++`.
- The `discovery` sub-stage exists precisely so the `level` stage stays **child-stages-only**: the level's discovery actions (the hero `image-gen` plus the five planning `llm-call`s — extractPalette/planRooms/planFeatures/topology/layout) live under `discovery` instead of directly on `level`, so `level`'s children are all sub-stages and never collide with the room sub-stages. `discovery` itself is action-only (all six via the `MAX+1` fallback, chronological: hero=0, planning=1..5). Wrapping same-kind leaf actions in a sub-stage is the cheap way to satisfy the invariant without hand-coordinating ranges.

Prefer this (a stage holds one kind of child) when the shape allows; reach for the hand-coordinated shared counter (the generation-stage example) only when a stage must hold both.

## Artifacts are a separate, action-scoped space

`recordArtifact` uses `input.sequence ?? 0`, scoped to one action's input/output payloads — it orders artifacts *within* an action, independent of the stage/action sequence space above. Multi-input actions (e.g. a future map-forge image call with several reference images) must thread explicit per-artifact sequences; single input+output actions can leave them at 0/0 since `direction` disambiguates.

## Rules of thumb

- Writing a new workflow audit stage? Decide up front: **all-actions, all-substages, or hand-coordinated.** If mixed, allocate one counter and give each writer a non-overlapping range.
- Never rely on write/wall-clock order — only `sequence` orders siblings.
- The `MAX+1` action fallback is "append after existing actions," not "append after existing children." It is blind to sub-stages.
- Cross-references: [[Review-Capture-Pipeline]] (what gets captured), [[Review-Service]] / [[Review-NodeTree-UI]] (the readers), [[Architecture-Determinism]] (workflow-local counters are deterministic; sequence allocation stays in workflow code, not activities), [[Temporal-Workflow-Activity-Boundary]].
