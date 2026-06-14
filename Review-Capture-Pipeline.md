---
Summary: How Map Forge and Asset Forge runs get captured for review. **One source:** the auditing decorators (`AuditingLLM`, `AuditingImageGenerator`, `AuditingRenderClient`) wrap the worker's outbound seams and write to the `agent_runtime` audit tree (stages · actions · action_artifacts). `AuditRunContext` (AsyncLocalStorage) tracks the open stage frame so each captured action attributes to the right room/attempt. `AssetPersister` links accepted assets via `recordAuditAssetOutput` (T-B7) and stamps a `verdict` (attempt history) on the artifact metadata; rejected candidates are linked the same way with `accepted=false`. Production Map Forge AND the integration tests both run through this path — the file-capture sibling (`StageManifestWriter` / `writeAttemptArtifacts` / `Capturing*`) was retired in PRs R-3 + R-4 (2026-06). [[Review-Service]] projects the resulting tree as-is into the tree-shaped wire contract the frontend consumes (`projectRunTree` / `projectNodeDetail`); the UI is [[Review-NodeTree-UI]].
Tags: #review #tooling #atlasforge #service-architecture #foundational
---

# Review Capture Pipeline

The review tool reviews *real* model output, so something has to run the real agent and persist every model decision in a reviewable shape. That "something" is the auditing-decorator pipeline — same code path in production runs and integration tests. This atom documents what gets recorded, how attribution works, and where the audit tree lives.

## One source: agent_runtime

Captured runs land in three tables under the `agent_runtime` schema (a separate physical DB from `atlasforge`):

| Table | Shape |
|-------|-------|
| `stages` | Tree of generation lifecycle frames: `run` → `phase` → `room` / `attempt`. Each row carries `name`, `stage_type`, `metadata` (`{ levelIndex, roomId, attemptIndex, … }`), status, and parent. |
| `actions` | Leaf events under a stage: `llm-call`, `image-gen`, `render`, `image-service`, `asset-persist`. Carries `action_type`, `metadata` (`{ promptName, provider, … }`), model/token/cost/latency telemetry. |
| `action_artifacts` | Per-action input/output payloads. `kind` ∈ {`text`, `asset`, `json`}; `direction` ∈ {`input`, `output`}; `asset_id` is the soft cross-DB ref to the main `atlasforge.assets` table. |

The `agent_runtime` DB is read-only from the review service ([[Review-Service]] holds the three-pool wiring). [[Architecture-Decoupling]] §3 (repository pattern, artifact handoff): the review service has no dependency on the worker package — the agent_runtime row shapes are the entire contract.

## Auditing decorators

Three production decorators wrap the agent's outbound seams. Each wraps a *factory* so the audit context is in scope when the inner client is constructed:

- **`AuditingLLMFactory` / `AuditingLLM`** (`apps/worker/src/agents/audit/AuditingLLM.ts`) — over the LLM seam (`packages/prompt-store/src/types.ts`). Per `chat()` call: one `llm-call` action against the currently-open stage, with the rendered prompt as an input `text` artifact and the raw response as an output `text` artifact (even on failure). `promptName` is read off `rendered.parameters` and stamped on `metadata`.
- **`AuditingImageGeneratorFactory` / `AuditingImageGenerator`** (`AuditingImageGenerator.ts`) — over the image-gen seam used for hero / concept / top-down generation. Records one `image-gen` action per generation with the rendered prompt as input, model + cost/latency telemetry on the action.
- **`AuditingRenderClient`** (`AuditingRenderClient.ts`) — over `IRenderClient`. Render snapshots are the one genuinely-new image kind the audit model introduces (everything else is an asset already), so this decorator MINTS an asset: it persists the rendered PNG as a `RenderSnapshot`-category asset (default-excluded from search) via the injected persister, then records a `render` action whose output artifact references that `asset_id`.

All three are no-ops when no run is active (`!isAuditActive()`), so consumers don't need a separate prod/test code path. Audit capture is wired in `apps/worker/src/index.ts` when `AUDIT_ENABLED=true` (production); the integration tests construct the decorators directly.

## AuditRunContext — the AsyncLocalStorage stage stack

`apps/worker/src/agents/audit/AuditRunContext.ts` is the one piece of mutable state in the capture pipeline, and it's deliberately ambient:

- **`withAuditRun(input, fn)`** opens the root stage for a generation and runs `fn` inside an `AsyncLocalStorage` scope. Async work inside `fn` inherits the scope; concurrent work in a sibling AsyncLocalStorage doesn't leak. `endRootOnExit` controls whether the root stage closes on return (production: yes; integration tests sometimes hand-close).
- **`withStage(input, fn)`** opens a child stage under the current top-of-stack frame. The frame holds the stage id, root id, and per-frame `nextActionSeq` / `nextChildSeq` counters. Throw inside `fn` → stage closes `status='failed'` and re-throws; success closes `completed`. The Map Forge agent wraps each phase, room, and attempt in `withStage`.
- **`recordAuditAction(input)`** records an action against the top-of-stack stage. The frame's `nextActionSeq++` MUST stay synchronous with the read (no await between): furnishings persist concurrently within one phase frame, and the single-threaded event loop only keeps sequences unique while the read-modify-write is atomic.
- **`recordAuditArtifact(input)`** attaches an artifact to a specific action (action-scoped, not stage-scoped).
- **`recordAuditAssetOutput(assetId, metadata)`** synthesizes the `asset-persist` action that owns an accepted/rejected asset (see below).
- **`markCurrentRunFailed()`** flips `state.failed` so `withAuditRun` closes the root stage `failed` even when the caller swallows the throw.

The stack is `state.stack` on the `AsyncLocalStorage` store — push on entry, pop in `finally`. Sequences (`nextActionSeq` / `nextChildSeq`) are per-frame, monotonic, and never re-used.

## Asset linkage (T-B7) — accepted + rejected

`AssetPersister.persist` is the asset-side seam. After the v2 (processed) asset is created in the main `atlasforge` DB, the persister calls `recordAuditAssetOutput(result.assetId, { accepted: true, verdict })` — this synthesizes an `asset-persist` action against the currently-open audit stage, with one output `asset` artifact carrying `asset_id` + metadata `{ accepted: true, verdict: AuditVerdict }`.

`AuditVerdict` (P-3) is the attempt history the review tool's `AttemptCard` renders: `attemptCount`, `finalEvaluation` (the whole `AgenticPipelineResult.evaluation`), `rejectedCandidates[]` (each with `attemptNumber`, `isHardReject`, the four eval classes, `failureMode`, `chromaLeakSeverity`, `feedback`, `refineResult`, `refinedPrompt`), `refinedPrompt`, `acceptedRefineResult`.

Rejected candidates persist via `AssetPersister.persistRejectedCandidates`, which calls `recordAuditAssetOutput(rejectAssetId, { accepted: false, attemptIndex })` per candidate. The reject→accepted grouping is resolved at *projection time* via `asset_relationships.relationship_type='rejected_candidate_for'` (`AuditAssetResolver.resolveRejectedParents`) — the audit tree's sequence order can't safely group concurrent persists.

The v1 raw image (pre-bg-removal) is NOT separately recorded; it's reached at projection time via `assets.parent_version_id` (`AuditAssetResolver.resolveRawParents`) and surfaced as `images['final-raw']`. Same "row absent → silently omit" policy as the other resolvers.

## Stage-tree correlation (the `_*` convention)

Per-room / per-attempt attribution flows through invoke-time metadata on `_*`-prefixed keys (`_room_id`, `_attempt_index`, `_promptName`). `LLMPrompt.invoke` and `ImageGenPrompt.invoke` mirror these onto `rendered.parameters` so the auditing decorator reads them without parsing slug strings. `_promptName` is system-reserved (set by the prompt itself, not by callers) and non-spoofable.

The decorators then either (a) stamp them on the action's `metadata` directly, or (b) rely on the ambient stage frame — Map Forge opens explicit `room` and `attempt` stages (`withStage({ stageType: 'room', metadata: { roomId } })` etc.), and the projector's `contextFor` walks up the stage tree at read time to derive `roomId` / `attemptIndex` for any action that didn't stamp them itself.

Breaking the convention silently mis-attributes captures (the historical "everything lands on `top-down-room-unknown`" bug).

## Map Forge whole-map final render

After the pipeline finishes (Map Forge handles the approve flow), `MapForgeAgent.renderFinalMap` opens a `final-render` phase stage and renders the compiled map via the (audit-wrapped) render client. The action carries no `roomId`, which is how the projector tells the whole-map final render apart from per-room attempt renders. `projectFinalMapUrl` resolves it to a signed URL and surfaces it as `MapForgeRunDetail.finalMapUrl`. The render is non-fatal — failure closes the `final-render` stage `failed` (preserving audit truth) but never fails the run.

## Asset Forge / bg-removal-retry

R-1b extended the same capture path to the standalone Asset Forge generation flow (`ImageGenerationServiceV2`), so bg-removal-retry runs land in the same `agent_runtime` tree with the same `asset-persist` linkage and verdict. No separate writer.

## Projection back to the wire contract

The review service projects `agent_runtime` rows as-is into the tree-shaped `RunTree` / `NodeDetail` wire contract the frontend consumes (see [[Review-Service]] for the wiring and `apps/review-service/backend/src/services/auditProjection.ts` — `projectRunTree` / `projectNodeDetail` — for the pure projection functions). The frontend renders the tree directly; the flatten-and-reconstruct path was deleted in the LangGraph redesign Phase 6 (2026-06).

## What was retired (R-3 + R-4 + R-5, 2026-06)

- `StageManifestWriter` (numbered per-stage JSON + `manifest.json`) — deleted.
- `writeAttemptArtifacts` (per-asset `verdict.json` + raw/processed/final images) — deleted.
- `CapturingLLM` / `CapturingImageGenerator` / `CapturingRenderClient` / `CapturingApplyFix` / `CapturingArtifactRepository` — deleted.
- `apps/worker/test-fixtures/**/__output__/` runtime trees — no longer produced; the auditing decorators are the sole capture path.
- `REVIEW_DB_ENABLED` flag — removed; the DB IS the source.

## Boundaries

- What reads the captured tree and serves it: [[Review-Service]].
- How a reviewer navigates a run: [[Review-NodeTree-UI]].
- The pipelines being captured: [[MapForgeAgent]], [[PhaseOrchestrator]], [[AgenticImageGenerationPipeline]], [[AssetVersioning-and-Relationships]] (for the v2→v1 raw link).
- Foundational rules cited: [[Architecture-Decoupling]] (DI, repositories, artifact handoff), [[Architecture-Determinism]] (audit truth = projector source of truth), [[Type-Discipline]] (narrow at the DB/JSON boundary).
