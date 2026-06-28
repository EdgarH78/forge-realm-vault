---
Summary: SUPERSEDED. The inner assess→refine→generate→evaluate retry loop (`apps/worker/src/AgenticImageGenerationPipeline.ts`) for AI asset images was DELETED in the ADK+Temporal migration — asset-forge image generation moved to [[Orchestrator-Service]] in Phase 1 (the worker's standalone handler went at the #583 cutover; the shared classes + this pipeline were deleted in #613 once map-forge no longer used them). The same assess→refine→generate→evaluate flow is now the Asset-Forge workflow (`assetForgeWorkflow`) over coarse ADK activities. Kept as a redirect.
Tags: #image-gen #retry-loop #superseded #atlasforge
---

# AgenticImageGenerationPipeline  *(superseded)*

> **Status: SUPERSEDED — code deleted.** `AgenticImageGenerationPipeline.ts` + the worker `IntentAssessor` / `PromptRefiner` / `ImageEvaluator` were removed (#583 handler cutover, #613 shared-class deletion). The live agent is **[[AssetForgeAgent]]** (the `assetForgeWorkflow` in [[Orchestrator-Service]]) — read that for the current implementation.

## What replaced it
- The bounded `maxAttempts = 3` **retry loop** is now the `assetForgeWorkflow` orchestration; each step is a coarse, idempotent **activity** ([[Temporal-Workflow-Activity-Boundary]]).
- `intentAssessor.assess` → **`assessIntent`** activity; `promptRefiner.refine` → **`refinePrompt`**; generate + post-process → **`generateAndProcess`** (Gemini direct gen + .NET image-service `/api/process`, persist-then-evaluate, idempotency-wrapped); `imageEvaluator.evaluate` → **`evaluateImage`**.
- The **refiner-owns-retry-strategy** rule carries over verbatim into the ADK agents: the refiner is the sole authority over `backgroundColor` / `backgroundRemovalMethod` / `useReferenceImage` (the three marker lines), the evaluator only classifies. The magenta-default + chroma-key + severity-gated retry policy is unchanged ([[ImageService]]).
- Map-forge consumes asset-forge as a **child workflow** per unique asset (fanned out from the dedup barrier in `mapForgeParentWorkflow`), replacing the old `AssetResolver`-closure call site.
- Per-asset credit deduction is now the `deductCredits` activity (authoritative, post-persist); exhaustion surfaces as a `CREDITS_EXHAUSTED` event rather than a thrown `InsufficientCreditsError`.
