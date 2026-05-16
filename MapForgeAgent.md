---
Summary: The concrete orchestrator for AI-generated map creation in `apps/worker/src/agents/MapForgeAgent.ts`. Three entry points (create / message / approve) drive a clarify→plan→approve→execute lifecycle. The planning side runs a 4-step prompt pipeline with a hero-image fan-out; the execution side hands off to [[PhaseOrchestrator]] over a 7-phase chain. Every state change is persisted as an artifact with parent-link lineage.
Tags: #map-forge #orchestrator #agent #ai-pipeline #atlasforge
---

# MapForgeAgent

## Construction

Instantiated in `apps/worker/src/index.ts` with: `promptStore`, `repo` (IMapForgeRepository), `artifactRepo`, `pubsub`, `storage`, `eventsTopic`, `assetRepository`, `assetFileRepository`, `assetGenerator` (closure wrapping the [[AgenticImageGenerationPipeline]]), `mapRepository`, `mapVersionRepository`, `mapLabelRepository`, `creditService`, and an optional `MapForgeAgentOptions.createRenderClient` seam for capture-wrapping in the Map Forge review tool.

## Three entry points

- **`handleCreate({generationId, sessionId, userId, mapName, prompt})`** — runs the clarify step (`map-forge-planner-clarify`). If it returns clarifying questions, save a partial contract `{summary, assumptions, questions, userConstraints?}` and wait. If it returns no questions, run the full 4-step planning pipeline inline.
- **`handleMessage({generationId, sessionId, content?, answers?})`** — wizard continuation. Restores `userConstraints` from the prior persisted contract, runs the 4-step pipeline with `clarification_answers` and/or `rejection_feedback`, supersedes the prior contract artifact, publishes `planning_contract` SSE.
- **`handleApprove({generationId, sessionId})`** — loads the latest `planning-contract` artifact, writes an `approved-plan` artifact, sets phase to `plan_approved`, then constructs the per-phase dep graph (AssetResolver, LlmVocabularyResolver, SelfEvaluator, FixPlanner, RoomRerollService, RenderClient, ApplyFix adapter) and invokes [[PhaseOrchestrator]] over the canonical 7-phase chain.

## The 4-step planning pipeline (`_runPlanningPipeline`)

Progress band 0.00 → 0.10. Every prompt is loaded via `promptStore.getLatestPrompt(...)`; missing prompts throw `ConfigurationError`; AJV-flagged responses throw `ValidationError`.

1. **Concept/Style** (`map-forge-planner-concept`) → `{summary, assumptions, concept, style{mood, wall, floor}}`.
2. **Levels** (`map-forge-planner-levels`) → array of `{id, name, description, purpose}`.
3. **2.5 Hero fan-out** (`_runHeroFanOut`) — per-level concept-hero image generation (`map-forge-concept-hero`) + palette extraction (`map-forge-palette-extract`), parallelized at `HERO_GEN_CONCURRENCY = 4`. Each successful level gets `heroUrl`, `palette[]`, and `materials{wall, floor, accent}` enriched in-place on the levels array; failures are non-fatal (null heroUrl, null palette, null materials — downstream consumers narrow on null). Per-level asset_files + assets rows are upserted for the hero PNG.
4. **Per-level rooms** (`map-forge-planner-rooms`) — invoked once per level with concept + level + current_level context.
5. **Features** (`map-forge-planner-features`) — door/window/stair placement over rooms.

## Planner-faithfulness

The clarify step extracts hard count caps `userConstraints = {maxLevels?, maxRooms?}` from the user prompt. These are:

- threaded into every prompt as a soft constraint,
- enforced by **truncation** after each step (defense-in-depth — keep the first N because the LLM tends to put the narratively primary item first),
- persisted on the planning contract so a subsequent `handleMessage` re-run picks them up,
- and copied onto the approved plan.

## The execution phase chain

`handleApprove` instantiates a fresh `PhaseOrchestrator` and runs:

```
topology → first-pass layout → top-down-reference (4 stages: gen, readout, re-layout,
   asset assembly) → compilation → self-eval-and-fix → validation → completion
```

Each phase is a `PhaseDefinition` factory (`createTopologyPhase(promptStore)`, etc.) — see [[PhaseOrchestrator]] for the runner contract and the per-phase progress bands.

## The ApplyFix adapter

`SelfEvalAndFixPhase` consumes a single `applyFix` instance, but `ApplyFix.deps.inventory` is per-room. The handleApprove closure constructs a **fresh `ApplyFix` per call** so each room's inventory is passed by reference (so `swap` actions can mutate `visual_description` in place). `userId` is resolved lazily on first apply so existing test mocks that skip `repo.getGeneration` still pass.

## Failure modes

- `InsufficientCreditsError` during execution → `status='completed'` with note "Credits exhausted — partial results returned" and a `credits_exhausted` SSE event. The user keeps whatever phases finished.
- Any other throw → `status='failed'`, an `error` event on the relevant phase, and the orchestrator stops.
- Hero gen / palette extraction failure for a single level → null fields, pipeline continues (D-07).
- AJV validation failure on any planning step → fatal `ValidationError`.

All state transitions write artifacts via `artifactRepo.saveArtifact` with `parentArtifactIds` linking back through the lineage so the [[AgentRuntimeSchema]] preserves a full graph of "what produced what".
