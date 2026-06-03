---
Summary: The `_*` invoke convention — how invoke-time metadata (`_room_id`, `_attempt_index`, `_promptName`, …) flows from agent call sites through `LLMPrompt.invoke` / `ImageGenPrompt.invoke` onto `rendered.parameters`, where the auditing decorators ([[Review-Capture-Pipeline]]) read it to attribute each captured call back to the right room / attempt / stage. Underscore-prefixed keys are stripped from the template substitution map (so they never pollute the template variable space) and copied onto a separate metadata slot. `_promptName` is system-reserved (set by the prompt itself, non-spoofable). Internal contract; ignore at your peril when adding a new agent that needs per-room/per-attempt traceability. The retired integration-test `Capturing*` helpers (`CapturingLLM`, `CapturingImageGenerator`, …) used this same channel — they were deleted in R-4 (2026-06) when the auditing decorators became the sole capture path.
Tags: #testing #atlasforge #foundational #integration-tests
---

# The `_*` Invoke Convention

Agent code routes per-call attribution (room, attempt, prompt identity) through invoke-time arguments without polluting the prompt template's variable space. The mechanism is implicit and easy to break — this atom documents the contract that every prompt invocation MUST follow if its capture should attribute correctly.

## The convention

Callers pass invoke-time metadata via `_*`-prefixed keys on the `invoke()` arguments object:

```typescript
await imageGenPrompt.invoke({
  // ─── prompt vars (consumed by the YAML template) ───
  category: 'top-down',
  reference_image_url: refUrl,
  // ─── system metadata (mirrored to parameters; not template-consumed) ───
  _room_id: roomId,
  _attempt_index: attempt,
});
```

`ImageGenPrompt.invoke` (and `LLMPrompt.invoke` — same mirror behavior) then:

1. Strips `_*` keys from the template-substitution map (so they do NOT pollute the template variable space).
2. Mirrors the `_*` keys onto `rendered.parameters`.
3. The audit decorator on the outbound seam reads `rendered.parameters` and records the call against the right room / attempt / stage in the `agent_runtime` tree.

Code that needs per-room/per-attempt routing — e.g. attributing `top-down-room-{roomId}` captures back to a room's wire stage — reads `parameters._room_id` rather than parsing slug filenames.

## What is non-spoofable

`_promptName` is **system-reserved**. It is set by the prompt's own `metadata.name` field in the YAML (loaded by [[prompt-store]] at construction time), not by `invoke()` callers. If a caller passes `_promptName` as an invoke var, it is silently dropped from the mirrored parameters — the system-set name wins.

This is load-bearing: the projector's `stageTypeForAction` maps `metadata.promptName` → wire stage type via `PROMPT_NAME_TO_STAGE_TYPE`. A spoofable `_promptName` would let any caller mis-classify which stage a call attributes to.

Everything else (`_room_id`, `_attempt_index`, future `_*` keys) is caller-stamped and trusted — the discipline lives at the call site.

## Who consumes it

After PRs R-3 + R-4 (2026-06), the consumers are the production auditing decorators ([[Review-Capture-Pipeline]]):

- **`AuditingLLM`** reads `parameters._promptName` and stamps it on the `llm-call` action's metadata.
- **`AuditingImageGenerator`** reads `parameters._promptName` (image-gen prompt name like `image-gen/walls/refine`) and the per-room `_room_id` to attribute captures.
- **`AuditingRenderClient`** doesn't go through a prompt, so it reads `RenderOptions.roomId` / `attemptIndex` directly (same conceptual channel, different transport).

The room/attempt context is *also* set on the audit stage tree itself via `withStage({ stageType: 'room', metadata: { roomId } })` etc., so the projector can derive it from the stage tree even when an action didn't stamp it. The two channels reinforce each other; either one alone is enough.

## Historical context — the retired Capturing\* wrappers

The convention was originally introduced for the integration-test `CapturingLLM` / `CapturingImageGenerator` / `CapturingRenderClient` / `CapturingApplyFix` / `CapturingArtifactRepository` helpers that wrote `apps/worker/test-fixtures/**/__output__/` capture trees for the review tool to read. Those wrappers were deleted in PR R-4 once the auditing decorators ([[Review-Capture-Pipeline]]) became the sole capture path — the integration tests now assert directly against the `agent_runtime` audit tree.

The convention itself outlived the wrappers. Any future capture consumer (production or test) reads `rendered.parameters` the same way.

## What can break attribution

The historical bug: a new prompt invocation that omits `_room_id` (or stamps it wrong) silently lands its captures on a `top-down-room-unknown` placeholder, and the review tool's per-room view shows mystery calls under no room.

The audit decorator can't enforce this — `_room_id` is a soft optional. Mitigations:

- **Lint surface:** when adding an LLM or image-gen invocation in a per-room loop, scan call sites for the convention.
- **Test surface:** the live Map Forge integration test (`map-forge.integration.test.ts`) asserts that per-room counts match the room count — a missing `_room_id` would show up as captures landing on the wrong stage.
- **Stage-tree backup:** Map Forge's room/attempt loops wrap their work in `withStage({ stageType: 'room' | 'attempt', metadata: { roomId, attemptIndex } })`, so the projector's `contextFor` derives correlation from the stage tree even when an action's own metadata is missing.

## Boundaries

- How the decorators record + correlate: [[Review-Capture-Pipeline]].
- The wire contract the captures project into: [[Review-Service]].
- The prompt-store seam that mirrors `_*` keys onto `rendered.parameters`: read `packages/prompt-store/src/types.ts` for the exact API.
