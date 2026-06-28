---
Summary: The `apps/orchestrator` service — a Temporal worker running ADK agents that **replaced** the worker's hand-rolled agent pipeline ([[MapForgeAgent]] / [[PhaseOrchestrator]] / [[AgenticImageGenerationPipeline]]) for map-forge and asset-forge. Both are **cut over** in dev/`main` (the worker's agent code is deleted — #611/#613); the remaining work is prod / Temporal Cloud (Phase 4). All-TypeScript: official `@google/adk` + the Temporal TS SDK. Pub/Sub stays both ingress (request topics → start / signal workflows) and egress (events topic, byte-identical envelopes) so the API↔UI contract is unchanged; the `agent_runtime` DB stays system-of-record, Temporal history is ops-only. See `.ai/plans/2026-06-09-adk-temporal-migration/`.
Tags: #service #orchestrator #temporal #adk #migration #atlasforge
---

# Orchestrator-Service

## Why it exists
The worker's agent pipeline was hand-rolled: a bespoke [[PhaseOrchestrator]] state machine, custom retry loops in [[AgenticImageGenerationPipeline]], manual resume-via-DB-state, and a web of auditing decorators (~32k LOC). It worked but was brittle and had no durable-execution story — a crash or deploy mid-generation lost in-flight work. The orchestrator **replaced** the *agents* with **ADK** (the agent loop, tool-calling, structured output, multimodal) and the *runtime* with **Temporal** (crash/upgrade-safe durable execution, child-workflow fan-out, mid-job upgrades); the worker's agent code (`apps/worker/src/agents/**`, the standalone image-gen handler, `AgenticImageGenerationPipeline`) is now **deleted** (#583 / #611 / #613). The slim worker survives for the non-agent plumbing (asset-ingest, map-export, asset-reprocess, Patreon credit grants + tier-reconciliation/retention crons — see [[Worker-Service]]).

It also hosts the [[CreditSystem]] Temporal surface: the `checkCredits`/`deductCredits` activities (advisory pre-flight vs authoritative post-persist) used by the generation workflows, and the daily credit-**expiry** Schedule (`expireCohortsWorkflow` → `burnExpiredCohorts`) that replaced the worker's expiry cron. All ledger I/O is activities-only so workflows stay deterministic ([[Temporal-Workflow-Activity-Boundary]]). *(Note: the orchestrator is not yet in the prod deploy pipeline, so the expiry Schedule doesn't run in prod until it ships — see project tracking.)*

## Topology (API contract unchanged)
- **Ingress** — the orchestrator consumes the SAME Pub/Sub request topics the worker did (`atlas-agent-requests`, `atlas-image-gen-requests`). A `create` starts a workflow keyed `workflowId = generationId`; `message` / `approve` become Temporal **signals** to the running workflow. The [[API-Service]] keeps publishing to those topics, unchanged.
- **Egress** — activities publish the SAME SSE event envelope (`{source, traceId: generationId, generationId, ...data, timestamp}`) to the existing events topic (`atlas-agent-events` / `atlas-image-gen-events`); the API relays it as SSE verbatim. See [[PubSub-Topology]].
- **Prod** runs against **Temporal Cloud** (managed, mTLS); **local dev** runs the Temporal stack in Docker Compose (ports 7233 gRPC / 8233 UI).

## Layout (boundary by location)
```
apps/orchestrator/src/
  worker.ts        # composition root: builds deps, registers createActivities(deps)
  workflows/       # DETERMINISTIC only — no I/O, no Date, no adk/genai imports
  activities/      # all non-determinism: ADK agents, clients, side-effects
    createActivities.ts  # the closure-DI factory
  shared/          # idempotency util + repository, event envelope builder
  db/  pubsub/     # agent_runtime pool, Pub/Sub client bootstraps (IO, coverage-excluded)
```
The contracts that govern this code live in [[Temporal-Workflow-Activity-Boundary]].

## System-of-record
The `agent_runtime` Postgres DB remains the authoritative store for generation status + the audit stage/action/artifact tree that the [[Review-Service]] reads. The audit recorder + retention repos were **extracted to the shared `@atlasforge/agent-runtime` package** so api / worker / orchestrator import it cleanly — resolving the worker's old cross-app `apps/api/src` import (cf. [[Worker-Service]]). Temporal's own event history is **internal durability only** — never read by the API. Anything that must survive a restart is an artifact, not just a Temporal variable.

## Mapping to what it replaced
| Was (`apps/worker`, deleted #611/#613) | Orchestrator |
|---|---|
| [[MapForgeAgent]] handleCreate/Message/Approve | `mapForgeParentWorkflow` + Pub/Sub→signal ingress |
| [[PhaseOrchestrator]] + the `*Phase` chain | nested child workflows (parent → `levelWorkflow` → `roomWorkflow` → `assetForgeWorkflow` → `compileRoom`) |
| [[AgenticImageGenerationPipeline]] | the Asset-Forge workflow + ADK agents (coarse activities) |
| `withConcurrency` manual fan-out | Temporal child-workflow fan-out (`executeChild` + `allSettled`) |

Pure modules (`DungeonIRCompiler`, validators, geometry kernels) were extracted **verbatim** into the shared `@atlasforge/map-ir` package — no runtime coupling, imported by the orchestrator activities.

## Status
**Phases 0–3 landed on `main`.** Phase 0 (Temporal stack + idempotency/credit/event/audit primitives), Phase 1 (Asset-Forge cut over — orchestrator owns `atlas-image-gen-requests`, #583), Phase 2 (Map-Forge nested workflow tree cut over — orchestrator owns `atlas-agent-requests`, #611), Phase 3 (worker agent code deleted #613; the live API+SSE integration harness rebuilt #614). Proven **live end-to-end**: a real map ran planning → images → compile → validate → commit and produced a committed map. **Remaining: Phase 4** (prod / Temporal Cloud) — the orchestrator is not yet in the prod deploy pipeline, so the worker-handler deletion must not ship to prod ahead of it. Full plan + live status: `.ai/plans/2026-06-09-adk-temporal-migration/README.md` (§0 tracker).
