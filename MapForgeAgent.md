---
Summary: The **map-forge agent** — the system that turns a text prompt into a committed multi-room battlemap. Its *implementation* moved to Temporal in the ADK+Temporal migration (it is NOT a class anymore — there is no `MapForgeAgent.ts`), but the agent itself is very much alive: it is the `mapForgeParentWorkflow` nested child-workflow tree (parent → level → room → asset → eval) plus the ADK-agent activities it drives, all in [[Orchestrator-Service]]. Lifecycle: clarify → upfront-charge → plan → **approve gate** → per-level hero/topology/layout → per-room reference/readout/vocabulary → **map-wide dedup barrier** → asset-forge fan-out → compile → record-only eval → fan-in → compile/validate/commit. `create`/`message`/`approve` arrive as Pub/Sub→Temporal signals; fan-out is `executeChild`+`allSettled`; durability is replay-determinism. See [[Temporal-Workflow-Activity-Boundary]].
Tags: #map-forge #orchestrator #agent #temporal #adk #atlasforge
---

# MapForgeAgent

The map-forge agent forges a whole battlemap from one user prompt. There is **no `MapForgeAgent` class** — the implementation moved to Temporal — but the agent is the live thing: a **nested workflow tree** in `apps/orchestrator/src/workflows/` driving a registry of **single-argument ADK + side-effect activities** (closure-DI'd in `createActivities.ts`). Workflow code is deterministic-only; every side effect is a proxied activity ([[Temporal-Workflow-Activity-Boundary]]).

## The workflow tree
```
mapForgeParentWorkflow              (clarify · upfront charge · concept/levels plan · approve gate)
├─ levelWorkflow      × N levels    executeChild + Promise.allSettled
│    hero→palette · planRooms→planFeatures · topology→layout
│    └─ roomWorkflow  × M rooms     genReferenceImage→readout→resolveVocabulary  → RoomCatalog (DISCOVERY ONLY)
│
│  ══ MAP-WIDE DEDUP BARRIER — fan-in of every room's uniqueAssets ══
│
├─ assetForgeWorkflow × K unique    one [[AssetForgeAgent]] child per deduped asset  → AssetVersionRef
│  ── compileRoom per room (pure: readout + resolved deduped assets) → furnished IRRoom ──
├─ roomEvalWorkflow   × room        renderRoom→evaluateRoom→planFix  (RECORD-ONLY, single pass)
└─ fan-in → compileMap → validateMap → commitMap → emit(completed, mapId)
```
The parent fans out three child types, each with the same idiom: call `executeChild(...)` N times **then** `await Promise.allSettled([...])` — children run concurrently and one failure can't sink the map (partial-results policy; a dropped child loses its furnishings, not the run). Child `workflowId`s key on the in-history `generationId` (`${generationId}:level-${i}`, `${generationId}:asset-${dedupKey}`, `${generationId}:eval-${roomId}`) for replay stability.

`roomWorkflow` is deliberately **discovery only** — it produces a `RoomCatalog` of `UniqueAssetRequest`s but does **not** generate assets or compile. Generation and `compileRoom` are hoisted up to the parent past the barrier, because rooms can't dedup against each other mid-flight.

## Lifecycle stages, in order
All in the parent unless noted.
1. **Clarify** — `assessMapIntent` (ADK) → `PlanningContract` (summary, assumptions, questions). Persisted (v1) + `emit(planning_contract)`.
2. **Clarify wizard** — `while contract.questions.length > 0`: `await condition(...)` for a `message` signal carrying `answers`; resolved answers supersede the contract (v++), re-persist, re-emit. A detailed prompt with no questions skips straight through.
3. **Upfront credit charge (PCF-1)** — post-clarify, **pre-approve** (planning runs before approve): `checkCredits`→`deductCredits` keyed `actionType:'map_forge_planning'`, `dedupeKey:'${generationId}:planning'`. Insufficiency → `emit(error, INSUFFICIENT_CREDITS)` + non-retryable throw.
4. **Plan** — `planConcept` → `MapConcept`; `planLevels` → `MapLevel[]` (truncated to `maxLevels`; empty → `NoLevelsError`).
5. **Level fan-out** (`levelWorkflow`) — per level: **hero image** (`genHeroImage`→`extractPalette`, both non-fatal `{ok:false}` on failure) · **topology** (`topology` → room graph) · **layout** (`layout` → geometry) · per-room **discovery** (`roomWorkflow`).
6. **Plan preview + approve gate** — `emit(planning_contract, buildPlanPreview(...))` (concept/levels/rooms/heroes, `questions:[]` — the UI's `hasPlanPreview` screen), **then** `await condition(() => approved)`, **then** `persistApprovedPlan` advances the generation to `plan_approved`. All paid asset generation sits *after* this gate — a never-approved run has been charged only for planning.
7. **Dedup barrier** — collect every fulfilled level's `roomCatalogs[].uniqueAssets`, first-wins dedup into `Map<dedupKey, UniqueAssetRequest>`. `dedupKey = category::canonicalKey::material::style`, so the same chair across N rooms generates once.
8. **Asset fan-out** — one [[AssetForgeAgent]] (`assetForgeWorkflow`) child per unique asset; then `resolveAssetInfo` reads each minted asset's dims/facets into the resolved set.
9. **Per-room compile** — `compileRoom` (pure activity) per readable room → furnished `IRRoom` from readout + resolved deduped assets.
10. **Eval (record-only, D-2e-10)** — one `roomEvalWorkflow` per room with a top-down `reference`: `renderRoom`→`evaluateRoom`→`planFix`. Single pass, **no applyFix / no re-gen** — returns the room byte-identical plus a recorded `{evaluation, recommendedFixes, disposition}`. A `RenderServiceError` → `accepted-no-render`.
11. **Commit tail** — `buildFullMapIr` merges level IRs + furnished rooms (drops empty levels, prunes dangling connections) → `compileMap` → `validateMap` (gate only — throws on unrecoverable structural errors) → `commitMap` (CREATE Map + version + `current` label). `emit(completed, mapId)`.

Graceful exits: no furnished rooms → `outcome:'no-rooms'`; `InsufficientCreditsError` in the tail → `failGeneration` + `outcome:'credits-exhausted'`.

## How it uses Temporal
- **Signals** (`workflows/signals.ts`) — the [[PubSub-Topology]] ingress routes a `map-forge` message by `action`: `create` → `client.workflow.start` (`workflowId = generationId`, `REJECT_DUPLICATE` reuse policy → re-delivery is a safe no-op); `message`/`approve` → `getHandle(generationId).signal(...)`. The workflow registers handlers: `approve` sets `approved=true`; `message` pushes onto an `inbox[]` queue.
- **Child fan-out + the one real barrier** — three `executeChild`+`allSettled` fan-outs; the dedup barrier is the single place a true barrier is required (the parent must see *all* rooms' assets before deduping).
- **Replay determinism** — every workflow file is I/O-free (no `Date.now`/`Math.random`/`@google/adk`/DB), enforced by ESLint (`no-restricted-imports`/`-properties` scoped to `workflows/**`) **and** a replay gate (`*.replay.test.ts` runs `Worker.runReplayHistory` against committed `__fixtures__/*.history.bin`). A determinism-breaking change throws `DeterminismViolationError` in CI. See [[Temporal-Workflow-Activity-Boundary]] §1.
- **Throttled image queue** — `genHeroImage`/`genReferenceImage`/`generateAndProcess` are proxied onto `GEMINI_IMAGE_TASK_QUEUE` ('gemini-image-gen'); the worker fail-closed splits them into a second rate-limited worker (`maxTaskQueueActivitiesPerSecond`) so they can *only* run throttled — the 429 defense (see [[AssetForgeAgent]]).
- **Idempotency** — `deductCredits` dedups on the Formance Idempotency-Key; `publishEvent` on `${generationId}:event:${seq}`; `commitMap` is the lone non-idempotent terminal → `maximumAttempts: 1`.

## Activities
ADK-agent activities (via `AdkAgentRunner` — `@google/adk` `LlmAgent`+`Gemini`, structured `run` or free-text `runText`): `assessMapIntent`, `planConcept`, `planLevels`, `planRooms`, `planFeatures`, `topology`, `layout`, `extractPalette`, `readout`, `resolveVocabulary`, `evaluateRoom`, `planFix`, `validateMap`. Side-effect activities: `genHeroImage`/`genReferenceImage` (Gemini direct gen + persist as ConceptArt/Intermediate), `compileRoom`/`compileMap` (pure IR transforms), `resolveAssetInfo`, `renderRoom`, `commitMap`, `persistPlanningContract`/`persistApprovedPlan`, `checkCredits`/`deductCredits`, `failGeneration`, `publishEvent`, and the audit recorder set.

## Egress / SSE
A workflow-local `emit(data)` increments a monotonic `seq` and calls the `publishEvent` activity (best-effort — a failed publish never fails the run; the **timestamp is stamped in the activity**, not the workflow). `eventsTopic` is injected by the ingress, never read from env. Event types: `planning_progress`, `planning_contract` (clarify contract early, plan preview at the gate), `completed` (carries `mapId`), `error` (`INSUFFICIENT_CREDITS`/`CREDITS_EXHAUSTED`/`MAP_BUILD_FAILED`). The envelope is byte-identical to the worker's, so the API SSE relay + UI are unchanged ([[Temporal-Workflow-Activity-Boundary]] §5).

## System-of-record + credits
The `agent_runtime` audit tree (best-effort, mirrors asset-forge) opens a `run` root + `planning` and `self_eval_and_fix` stages; eval children nest `room → attempt` sub-trees; every call rides `auditTry` (swallows failures) and `closeAuditTree` is idempotent on every terminal/throw path. Two charge points: upfront planning (once, pre-approve) and per-asset (each [[AssetForgeAgent]] child, post-persist). Pure modules (`DungeonIRCompiler`, validators, `furnishingGeometry`) are imported verbatim from `@atlasforge/map-ir`.

The map-forge agent's predecessors — the deleted worker classes — are [[PhaseOrchestrator]] (its sequential phase runner, now the workflow structure) and the per-image inner loop [[AgenticImageGenerationPipeline]] (now [[AssetForgeAgent]]). Contrast the open-ended [[DeepAgent]]: map-forge is a *fixed* pipeline, not a replanning loop.
