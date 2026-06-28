---
Summary: The **asset-forge agent** — the system that turns one furnishing/asset request into a finalized, background-removed asset image. Like [[MapForgeAgent]] it is NOT a class (no `AssetForgeAgent.ts`); it is the `assetForgeWorkflow` durable control loop (`apps/orchestrator/src/workflows/assetForgeWorkflow.ts`) + its activities + three ADK agents (`intent_assessor` / `prompt_refiner` / `image_evaluator`) run through one `AdkAgentRunner`, with the pure gate/rank/marker logic in `@atlasforge/asset-forge-core`. It is a bounded **assess → [refine → generate → evaluate] ×≤5 → persist → charge** retry loop, ported byte-faithfully from the worker's old `AgenticImageGenerationPipeline`. Runs two ways: standalone (image-gen Pub/Sub ingress) and as a **child workflow** fanned out from [[MapForgeAgent]]'s dedup barrier. See [[Temporal-Workflow-Activity-Boundary]].
Tags: #asset-forge #image-gen #orchestrator #agent #temporal #adk #atlasforge
---

# AssetForgeAgent

The asset-forge agent generates one asset image from a request and finalizes it (background removed, persisted, credited). There is **no `AssetForgeAgent` class** — the agent is the `assetForgeWorkflow` (deterministic control loop) + its activities + the ADK agents it drives. The workflow file's header calls it a port of the worker's `AgenticImageGenerationPipeline.generate()`; the agent persona lives in the prompt store + `AdkAgentRunner`, not in a TS class. This is the inner loop; [[MapForgeAgent]] is the outer one.

## The control loop
`assess → [refine → generate → evaluate]×≤maxAttempts → rank → persist → charge`. `maxAttempts` defaults 3, ceiling 5. Each attempt:
```
refinePrompt   (ADK prompt_refiner, free-text + marker lines)
generateAndProcess  (Gemini direct gen → gated bbox → .NET image-service → persist candidate)
evaluateImage  (ADK image_evaluator, structured output → ImageEvaluation)
→ rank candidate (pure: soft beats hard; fewer failing axes wins)
→ clean pass? accept + break.  else feed feedback into next refine.
```
- **Soft early-out**: after 2 attempts, if the best candidate isn't a hard-reject, break (good enough).
- **Best-candidate ranking** (`@atlasforge/asset-forge-core`): every attempt is pushed to `candidates[]`; `bestIdx`/`acceptedIdx` track the winner. At loop end `chosenIdx = acceptedIdx >= 0 ? acceptedIdx : bestIdx`. A non-clean run still finalizes the best candidate (`accepted:false`).
- **Output**: `{kind:'completed', assetId, attemptCount, accepted}` or `{kind:'duplicate'}`. If every attempt deduped → `{kind:'duplicate'}` (a non-failure); if all failed for other reasons → non-retryable `AllAttemptsFailedError`. The chosen candidate is finalized; the rest pass to `persistAsset` as `rejected[]` (linked for review).

The workflow is deterministic — ranking/gates/feedback/marker-parsing are pure `@atlasforge/asset-forge-core` imports; all I/O is proxied activities ([[Temporal-Workflow-Activity-Boundary]] §1). Refs (never image Buffers) cross the workflow/activity boundary and thread between attempts.

## Activities
**ADK-agent activities** (each runs a `prompt_*`/`image_*` agent via `IAgentRunner`; all fall back to DEFAULTS on failure rather than throw — durability parity with the old worker):
- `assessIntent` — `intent_assessor`, structured output → `IntentAssessment` (category, art style, facets); applies caller `overrides`.
- `refinePrompt` — `prompt_refiner`, **free-text** (`runText`, no output schema — the YAML emits marker lines); loads `image-gen/<category>/refine` → falls back `image-gen/refine`; threads the prior raw image as a vision part on retry.
- `evaluateImage` — `image_evaluator`, structured output → `ImageEvaluation`; resolves refs→bytes **inside** the activity in `[raw, processed]` order (load-bearing for the chroma-leak compare); coerces every axis against anti-injection allowlists.

**Side-effect activities** (throw → Temporal retries):
- `generateAndProcess` — the gen pipeline: prompt resolve → bg/method select → reference threading → **Gemini direct gen** → **gated bbox** (Gemini-Vision bbox called *only* for `sam`/`bbox-crop`; chroma-key skips it) → **.NET image-service** processing → **persist candidate** as a Drafted v1(raw)/v2(processed) version chain. Returns refs `{rawRef, processedRef}` or `{kind:'duplicate'}`. **Persist-then-evaluate**: it persists and returns *refs*; eval/refine consume refs, never bytes.
- `persistAsset` — loop-end finalize of the chosen candidate (`finalizeAsset` status transitions on the already-Drafted rows); pre-signs a 24h thumb URL; builds the `RecorderAssetAuditSink`.
- `resolveAssetInfo` — reads a minted asset's dims + `size`/`wall_attached` facets for [[MapForgeAgent]]'s `compileRoom` (`{ok:false}`, never throws).
- `checkCredits` / `deductCredits` · `publishEvent` · the audit recorder set.

## How it uses Temporal
- **Idempotency** (`shared/idempotency.ts`: get → run `fn` → `putIfAbsent` → re-read) keys: `generateAndProcess` → `${generationId}:attempt:${n}:gen`; `persistAsset` → `${generationId}:persist`; `publishEvent` → `${generationId}:event:${seq}`. `deductCredits` does **not** use the util — it passes `dedupeKey` as the Formance ledger `Idempotency-Key` (the source-of-truth no-op).
- **Retry policy** — `generateAndProcess` is bounded to `maximumAttempts: 2` (every retry re-bills Gemini — the *workflow loop* is the real retry budget); persist/credits `5`, events/audit `3`, ADK agents `3`.
- **Throttled queue** — `generateAndProcess` runs on `GEMINI_IMAGE_TASK_QUEUE` ('gemini-image-gen'). `worker.ts` pulls it (+ `genHeroImage`/`genReferenceImage`) out of the default worker and spins a second activity-only worker with the throttle spread in. Throttle is opt-in env: `GEMINI_IMAGE_MAX_RPS` → `maxTaskQueueActivitiesPerSecond` (server-enforced, the 429 defense), `GEMINI_IMAGE_MAX_CONCURRENT` → `maxConcurrentActivityTaskExecutions`. Unset → SDK default (unlimited), never a hard cap.

## Two invocation paths
- **(a) Standalone** — the image-gen ingress subscribes `atlas-image-gen-requests`, verifies the JWT, trusts the verified `sub` over body `userId`, and `client.start('assetForgeWorkflow', { workflowId: generationId, ... })` (`REJECT_DUPLICATE` → re-delivery is a safe no-op).
- **(b) Child of map-forge** — [[MapForgeAgent]]'s map-wide dedup barrier fans out one `executeChild(assetForgeWorkflow, ...)` per unique `dedupKey` (the same chair in N rooms generated once); child `generationId = ${parent}:asset:${dedupKey}`, threaded `levelPalette`/`levelMaterials` + per-asset `creditCost`. A failed/duplicate child drops its furnishings (`{ok:false}`) rather than sinking the map.

## Refiner-owns-strategy contract
The refiner is the **sole authority** over the three strategy markers, emitted as trailing free-text lines and parsed by `parseRefineMarkers` (`@atlasforge/asset-forge-core`):
```
BACKGROUND_COLOR: #FF00FF
BACKGROUND_REMOVAL_METHOD: chroma-key-flood
USE_REFERENCE_IMAGE: false
```
Ported **verbatim** from the worker's `PromptRefiner` so strategy decisions are byte-identical. `generateAndProcess` consumes them as authoritative (refiner color wins over the category default; refiner method wins over the chroma-key default; `useReferenceImage` gates prior-raw reference threading). The **evaluator only classifies** — it owns no strategy. Severity-gated retry is pure (`gates.ts`): `isHardReject` (wrong perspective per-category, or `wrong-category`) retries off-budget; `shouldRetryForSoftFail` retries only on perspective, categoryFit, or `chroma-leak` at `major`/`unusable` severity — styleConsistency + completeness are advisory-only. Allowlists: `ALLOWED_BG_COLORS = CHROMA_BG_HEXES` (magenta default; gray dropped), `ALLOWED_REMOVAL_METHODS = {sam, chroma-key, chroma-key-flood}`. See [[ImageService]].

## Credits + audit + persistence
`checkCredits` is an advisory pre-flight; the **authoritative** charge is `deductCredits` **after** `persistAsset` (asset rows stay finalized on insufficiency — the worker end-state), idempotent via ledger `dedupeKey` ([[CreditSystem]]). The `agent_runtime` audit tree opens a `run` root + one `generation` stage; `persistAsset`'s `RecorderAssetAuditSink` records one `asset-persist` action per asset with a verdict artifact (the metadata the review projector keys on). A persisted candidate is an `AssetVersionRef = {assetId, version}`; the chosen v2 is promoted to its target status. **`DuplicateCandidate` is a Result, not a throw** — `persistCandidate` returns null on the `(owner, sha256)` dedup; `generateAndProcess` maps it to `{kind:'duplicate'}` (throws aren't replay-stable, and the idempotency wrapper stores Results).

This is the live replacement for the worker's deleted [[AgenticImageGenerationPipeline]]; the durability/grain rules are [[Temporal-Workflow-Activity-Boundary]].
