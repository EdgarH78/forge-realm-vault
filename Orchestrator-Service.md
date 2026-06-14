---
Summary: The `apps/orchestrator` service — a Temporal worker running ADK agents that is progressively replacing the worker's hand-rolled agent pipeline ([[MapForgeAgent]] / [[PhaseOrchestrator]] / [[AgenticImageGenerationPipeline]]) for map-forge and asset-forge. All-TypeScript: official `@google/adk` + the Temporal TS SDK. Pub/Sub stays both ingress (request topics → start / signal workflows) and egress (events topic, byte-identical envelopes) so the API↔UI contract is unchanged; the `agent_runtime` DB stays system-of-record, Temporal history is ops-only. In-progress migration — see `.ai/plans/2026-06-09-adk-temporal-migration/`.
Tags: #service #orchestrator #temporal #adk #migration #atlasforge
---

# Orchestrator-Service

## Why it exists
The worker's agent pipeline was hand-rolled: a bespoke [[PhaseOrchestrator]] state machine, custom retry loops in [[AgenticImageGenerationPipeline]], manual resume-via-DB-state, and a web of auditing decorators (~32k LOC). It works but is brittle and has no durable-execution story — a crash or deploy mid-generation loses in-flight work. The orchestrator replaces the *agents* with **ADK** (the agent loop, tool-calling, structured output, multimodal) and the *runtime* with **Temporal** (crash/upgrade-safe durable execution, child-workflow fan-out, mid-job upgrades). The slim worker survives for the non-agent plumbing (map-export, asset-reprocess, Patreon credit grants + tier-reconciliation cron — see [[Worker-Service]]).

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
The `agent_runtime` Postgres DB remains the authoritative store for generation status + the audit stage/action/artifact tree that the [[Review-Service]] reads. The audit recorder (`RunAuditRecorderPostgres`) is being **extracted to a shared package** so api / worker / orchestrator share it cleanly — a deliberate departure from the worker's cross-app `apps/api/src` import (cf. [[Worker-Service]] §Cross-package imports). Temporal's own event history is **internal durability only** — never read by the API. Anything that must survive a restart is an artifact, not just a Temporal variable.

## Mapping to what it replaces
| Today (`apps/worker`) | Orchestrator |
|---|---|
| [[MapForgeAgent]] handleCreate/Message/Approve | parent workflow + Pub/Sub→signal ingress |
| [[PhaseOrchestrator]] + the `*Phase` chain | nested child workflows (parent → level → room → asset) |
| [[AgenticImageGenerationPipeline]] | the Asset-Forge workflow + ADK agents (coarse activities) |
| `withConcurrency` manual fan-out | Temporal child-workflow fan-out |

Pure modules (`DungeonIRCompiler`, validators, geometry kernels) are reused verbatim — they have no runtime coupling.

## Status
Phase 0 (foundations) in progress: local Temporal stack + worker skeleton landed; idempotency primitive + `agent_runtime` DB infra landed; PubSubEventPublisher egress + the closure-DI factory landed. ADK was de-risked **live** (structured output + multimodal image input + tool calling all confirmed against Gemini), so the build-on-ADK decision is validated. Migration order: asset-forge first → map-forge calls it as a child workflow → delete the worker's agent code. Full plan + live status: `.ai/plans/2026-06-09-adk-temporal-migration/README.md` (§0 tracker).
