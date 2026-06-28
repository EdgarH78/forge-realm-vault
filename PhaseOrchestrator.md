---
Summary: SUPERSEDED. The deterministic sequential phase runner (`apps/worker/src/agents/PhaseOrchestrator.ts`) that drove [[MapForgeAgent]]'s execution side was DELETED in the ADK+Temporal migration (#613). Its job — running a fixed phase chain in order, dual-publishing progress on Pub/Sub + a `generation_events` row, bailing on first failure, re-throwing `InsufficientCreditsError` — is now Temporal-native: the fixed chain became nested child workflows (parent → level → room → asset), progress is the `publishEvent` activity, and failure/retry is durable-execution-native. Kept as a redirect; see [[Orchestrator-Service]] / [[Temporal]].
Tags: #map-forge #phase-state-machine #superseded #atlasforge
---

# PhaseOrchestrator  *(superseded)*

> **Status: SUPERSEDED — code deleted.** Removed with the `apps/worker/src/agents/` tree in #613. Its role — sequencing the map-build phases — folded into the live **[[MapForgeAgent]]** workflow tree (`mapForgeParentWorkflow`) in [[Orchestrator-Service]]; read that for the current implementation.

## What replaced it
- The rigid `{name, progressStart, progressEnd, run}` phase **sequence** (fixed at the call site, no replan — the rigid counterpart to [[DeepAgent]]) is now the **deterministic workflow** code itself: `mapForgeParentWorkflow` and its child workflows encode the order, and Temporal **replays** them for durability ([[Temporal]]). The canonical chain (topology → layout → top-down-reference → compile → eval → validate → commit) survives as workflow structure + activities.
- **Progress dual-publish** (Pub/Sub SSE + a `generation_events` DB row, same `{source, traceId, generationId, …}` envelope) is now the `publishEvent` activity; the phase transition is the system-of-record `updatePhase`. The DB write is still the truth, Pub/Sub still the liveness signal ([[PubSub-Topology]]).
- **`InsufficientCreditsError` graceful handling** is now an `error` event with `code: INSUFFICIENT_CREDITS` / `CREDITS_EXHAUSTED` emitted from the workflow.
- **First-failure bail (sequential, no rollback)** is Temporal's native activity-failure → workflow-failure propagation with bounded retries; surviving artifacts still persist, and re-approve still regenerates from the latest planning contract.

The contracts that govern the replacement are [[Temporal-Workflow-Activity-Boundary]].
