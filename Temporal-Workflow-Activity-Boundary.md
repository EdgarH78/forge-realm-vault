---
Summary: The load-bearing rules for [[Orchestrator-Service]] code. Temporal workflows are deterministic skeletons; all non-determinism (LLM/ADK calls, DB, Pub/Sub, render, time, randomness) lives in activities. Enforced by location — `src/workflows/` vs `src/activities/`. Five invariants: (1) the determinism boundary, (2) coarse activity grain — one ADK agent = one activity, (3) the closure-DI factory — deps captured at Worker construction, only serializable args cross the proxy, (4) idempotent side-effects via an idempotency key, (5) a byte-compatible egress event envelope. Specialises [[Architecture-Determinism]] + [[Architecture-Decoupling]] for the Temporal runtime.
Tags: #orchestrator #temporal #determinism #idempotency #best-practices #atlasforge
---

# Temporal-Workflow-Activity-Boundary

These are the rules every contributor to [[Orchestrator-Service]] must hold. They specialise the foundational atoms for Temporal.

## 1. The determinism boundary
Temporal **replays** workflow code to reconstruct state, so workflow code must be deterministic: no I/O, no `Date.now()` / `new Date()` / `Math.random()`, no DB / Pub/Sub / HTTP, and no `@google/adk` or `@google/genai` imports. Enforced **by location** — workflow code lives in `src/workflows/`, everything non-deterministic in `src/activities/`. A workflow's only side-effects flow through `proxyActivities`. This is [[Architecture-Determinism]] applied to the Temporal runtime. A **replay test** (re-running new workflow code against recorded histories) is the CI guard that protects the mid-job-upgrade guarantee — it must gate any workflow-code change.

## 2. Coarse activity grain
Each **ADK agent — including its bounded internal loop** (e.g. refine→generate→evaluate, cap 3) — is **one Temporal activity**. Child workflows handle the per-level / per-room / per-asset fan-out. This matches the design diagrams (subgraph box = workflow, agent node = activity), keeps the Temporal event history small, and uses ADK's own loop as intended. A mid-activity crash replays the whole agent — accepted as rare (Decision D7). Documented escape hatch, unbuilt unless reliability data demands it: split the expensive image-gen / bg-removal / render HTTP calls into their own activities.

## 3. Closure-DI activity factory
Temporal proxies activities **by name** and can carry only **serializable** args across the workflow→activity boundary. Non-serializable infra (the Pub/Sub client, DB repositories, secrets) therefore **cannot be an activity argument** — it is captured in a closure at Worker construction, per Temporal's `createActivities(deps)` pattern:

```ts
export const createActivities = (deps: ActivityDeps) => ({
  publishEvent: (input) => publishEvent(deps.topicResolver, deps.idempotencyRepository, input),
});
export type Activities = ReturnType<typeof createActivities>;
```

- `worker.ts` is the **composition root** — it builds the real deps and registers `createActivities(deps)`.
- Workflows proxy `proxyActivities<Activities>()`, importing `Activities` **type-only** so infra never enters the sandbox.
- The underlying impl functions keep explicit-deps signatures so they stay unit-testable; the factory only injects.

Registered activities take a single serializable input; deps come from the closure. See [[Architecture-Decoupling]] §1 (DI) / §2 (interface-first).

## 4. Idempotent side-effects
Temporal **auto-retries** activities, so any activity that **deducts credits, persists an asset, or publishes a user-visible event** must be idempotent — keyed `generationId:…:step` through the idempotency repository (`agent_runtime.idempotency_keys`; flow: get → run `fn` → `putIfAbsent` → re-read). A retry returns the stored result without re-running the effect; a void effect stores JSONB `null`. The util's one limit: a *truly concurrent* duplicate may run `fn` twice (one stored winner) — exactly-once still requires the **effect itself** to be safe. Credit deduction is the canonical example: it doesn't lean on this util at all — the `deductCredits` activity passes the `dedupeKey` straight to the Formance ledger as its **`Idempotency-Key`**, so the ledger no-ops a replayed spend at the source of truth ([[CreditSystem]]). Decision D8.

## 5. Egress contract (API/UI unchanged)
Live progress events are published to the **existing** Pub/Sub events topic with the byte-compatible envelope `{source, traceId: generationId, generationId, ...data, timestamp}`; the **timestamp is stamped inside the activity** (workflows can't call `Date`, and a replay must not re-stamp — the idempotency wrapper skips the publish). This keeps the [[API-Service]] SSE relay + UI untouched (Decision D5). The DB half of the worker's old dual channel (the `agent_runtime` event row + audit tree) is written by the audit recorder, not the publisher — see [[Orchestrator-Service]] §System-of-record.

## Testing
- **Activity unit tests** — call the impl with faked deps (faked topic resolver / idempotency repository); assert structured output + side-effects fire exactly once.
- **Workflow tests** — `@temporalio/testing` time-skipping env with **mocked activities**; assert fan-out counts, signal handling (clarify/approve), and retry caps deterministically — no live model calls.
- **Replay tests** — the determinism guard from §1.
- Bootstrap/IO files (`worker.ts`, `db/pool.ts`, `pubsub/client.ts`) are coverage-excluded; everything testable meets the orchestrator threshold.
