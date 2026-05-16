---
Summary: The inner assess→refine→generate→evaluate retry loop for AI asset images. Bounded at `maxAttempts = 3` by default. Hard-gate / soft-gate severity scoring decides whether each candidate is accepted, retried, or stored as a fallback. The refiner is the sole authority over `(backgroundColor, backgroundRemovalMethod, useReferenceImage)` on retry; the evaluator only classifies. Called from [[MapForgeAgent]] via the AssetResolver closure during the top-down-reference phase ([[PhaseOrchestrator]] chain).
Tags: #image-gen #retry-loop #agent #pipeline #atlasforge
---

# AgenticImageGenerationPipeline

## Identity

`AgenticImageGenerationPipeline` (`apps/worker/src/AgenticImageGenerationPipeline.ts`) implements `IAgenticImageGenerationPipeline`. Constructed with `(intentAssessor, promptRefiner, imageEvaluator, promptStore, eventTopic, pubsub, directGenerator, processingClient, geminiVision?)`. The generation path is `DirectGeminiImageGenerator → Gemini 2.5 Flash`; post-processing is `ImageServiceProcessingClient → .NET image-service /api/process` (bg removal + resize + WebP). The .NET service is NOT in the generation path.

## The loop

```
generate(req): AgenticPipelineResult
  intent = await intentAssessor.assess(req.prompt, req.overrides)
  loop while totalAttempts < maxAttempts:
    refineResult = await promptRefiner.refine({ userPrompt, intent,
        previousRefinedPrompt, feedback, levelPalette, levelMaterials,
        priorAttempt })                                   // attempt N≥2 carries priorAttempt
    {rawBuffer, processedBuffer} = await generateImage(intent, refineResult, ...)
    evaluation = await imageEvaluator.evaluate({
        imageBuffer: processedBuffer, rawImageBuffer: rawBuffer,    // both — for chroma-leak detection
        intent, userPrompt, refinedPrompt })
    priorAttempt = buildPriorAttempt(refineResult, evaluation, rawBuffer)
    allCandidates.push({ raw, processed, evaluation, refineResult, attemptNumber, isHardReject })
    track bestCandidate (soft beats hard; fewest failing axes wins)
    if !hardReject && !shouldRetryForSoftFail(evaluation, category): return accepted
    if totalAttempts > 2 && bestImage && !bestIsHardRejected: break  // good enough
  return bestCandidate (with rejectedCandidates[] for audit)
```

## Refine — three marker lines

`IPromptRefiner.refine(...) → RefineResult { refinedPrompt, backgroundColor?, backgroundRemovalMethod?, useReferenceImage? }`. The refine YAML instructs the LLM to emit three marker lines (`BACKGROUND_COLOR:`, `BACKGROUND_REMOVAL_METHOD:`, `USE_REFERENCE_IMAGE:`) which the parser strips out of `refinedPrompt`. Each is `undefined` when absent, malformed, or out-of-allowlist. On retry (`priorAttempt` set), the refiner sees the prior strategy + the evaluator's `failureMode` + the prior raw PNG multimodally, and picks a corrective combination.

## Generate — split pipeline

`generateImage(intent, refinedPrompt, request, refineDecision, priorRawForReference)`:

1. Resolve prompt: subcategory → category → `image-gen/generic/image`.
2. **Background color**: `refineDecision.backgroundColor` wins; default magenta (`#FF00FF`) for most, white (`#FFFFFF`) for `DARK_ASSET_CATEGORIES`, gray (`#808080`) only as chroma-leak fallback. Magenta + chroma-key has zero hue overlap with stone/iron/wood and is the production default since 2026-04-30.
3. **Removal method**: `refineDecision.backgroundRemovalMethod` wins; YAML default fills in; chroma-key is paired with magenta automatically.
4. **Reference image**: `useReferenceImage=true` + prior raw + no user-supplied `referenceImageBase64` → thread prior raw multimodally to Gemini.
5. Call `directGenerator.generate(...)` → raw PNG.
6. Conditional `geminiVision.detectForegroundBoundingBox(...)` — only when method is `sam` or `bbox-crop`; chroma-key skips this call.
7. `processingClient.process(rawBuffer, mime, processingOptions)` → processed WebP.

## Evaluate — severity-classified gates

`IImageEvaluator.evaluate(...) → ImageEvaluation { perspectiveClass, categoryFitClass, styleConsistencyClass, completenessClass, failureMode, chromaLeakSeverity, reasons[], feedback }`. Per-category rules in `CATEGORY_GATE_RULES` define `hardRejectPerspectives`, `passPerspectives`, `requireStyleConsistency`, `requireCompleteness`:

- **`isHardReject`**: perspective ∈ hardRejectPerspectives OR `categoryFit === 'wrong-category'`. Immediate retry, doesn't count toward the soft budget.
- **`shouldRetryForSoftFail`** (260502-iyj): wrong perspective, wrong category, OR `failureMode === 'chroma-leak'` with severity `major` / `unusable`. Style + completeness are **advisory only** post-iyj — they appear in feedback but do not gate retry on their own.
- **`chromaLeakSeverity`**: `minor` accepts, `major` retries with bg-color swap, `unusable` retries with method swap. Drives the refine-YAML PRIOR ATTEMPT CONTEXT block.

## Candidate ranking

`allCandidates` accumulates every attempt (raw + processed + evaluation + refineResult). The "best" tracker prefers a soft-only candidate over any hard-rejected one; among same class, fewer failing axes wins. If all attempts are hard-rejected, the loop still returns the candidate with the fewest hard-reject axes — "a slightly wrong perspective is better than no image at all." The accepted attempt is filtered out of the returned `rejectedCandidates[]`.

## Call site

`MapForgeAgent.handleApprove` constructs an `AssetResolver` wrapping the `assetGenerator` closure (`apps/worker/src/index.ts`). The TopDownReferencePhase's Stage 4 and `ApplyFix.regenerate` / `swap` actions both call `assetResolver.resolve(manifest, userId)` which dispatches through this pipeline for any asset not already in the catalog. Credits are deducted per resolved asset; `InsufficientCreditsError` propagates up to [[PhaseOrchestrator]] and is re-thrown to [[MapForgeAgent]] for graceful partial-results handling.
