---
Summary: How Temporal — the durable workflow runtime under [[Orchestrator-Service]] — actually works, in AtlasForge terms. A Worker polls a task queue and executes Workflows (deterministic, replayed orchestration) and Activities (non-deterministic side-effects, auto-retried); a Client starts and signals workflows. Activities are **registered by passing an object to `Worker.create({ activities })`** and **invoked by name** — a workflow's `proxyActivities<T>()` is matched to the registered object's keys at RUNTIME, not typechecked. Identity is `workflowId`; durability comes from a replayed event history. The determinism rules and our conventions on top of this live in [[Temporal-Workflow-Activity-Boundary]].
Tags: #temporal #runtime #workflow #activity #worker #atlasforge
---

# Temporal

The durable-execution runtime under [[Orchestrator-Service]]. This atom is the *mechanics*; the rules we hold on top of it are [[Temporal-Workflow-Activity-Boundary]].

## The four moving parts
- **Workflow** — deterministic orchestration code (the skeleton: fan-out, signals, retry loops). Temporal **replays** it to reconstruct state, so it must be deterministic (no I/O, no `Date`, no randomness). Lives in `src/workflows/`.
- **Activity** — a plain async function doing the non-deterministic work (LLM/ADK call, DB, Pub/Sub, HTTP). Auto-retried on failure. Lives in `src/activities/`.
- **Worker** — a long-running process that **polls a task queue** and executes the workflow + activity tasks it receives. No inbound HTTP — like [[Worker-Service]] it polls outbound. Constructed in `worker.ts`.
- **Client** — starts workflows and sends them signals. In the orchestrator the [[PubSub-Topology]] ingress adapter is the client: a `create` request → `client.workflow.start(...)`, a `message` / `approve` → `handle.signal(...)`.

## Registration & proxying (how an activity actually gets called)
This is the part that trips people up. There are three distinct roles:

| Piece | Role |
|---|---|
| `createActivities(deps)` (`src/activities/createActivities.ts`) | **Builds** the activity object `{ greet, publishEvent }`, injecting deps via the closure ([[Temporal-Workflow-Activity-Boundary]] §3) |
| `Worker.create({ activities: createActivities(deps) })` (`worker.ts`) | **Registers** them — Temporal reads the object's **keys** as the activity names |
| `export { createActivities }` (`activities/index.ts`) | Just **re-exports** the factory for import; registers nothing |

A workflow never imports an activity's implementation. It declares the ones it needs with `proxyActivities`:

```ts
const { greet } = proxyActivities<Activities>({ startToCloseTimeout: '1 minute' });
```

`Activities` is a **type only** (`ReturnType<typeof createActivities>`), imported `import type` — erased at compile, present purely for type-checking/autocomplete. At runtime, calling `greet(name)` enqueues an activity task named `"greet"`; the Worker looks `"greet"` up in its registered `activities` object and runs it.

**Matching is by name, at runtime — NOT typechecked.** Temporal does not verify that the Worker registered every activity a workflow proxies. Consequences:
- A test Worker can register a **subset** (`{ greet: mock }`) and still drive a workflow that only calls `greet` — this is why `hello.test.ts` works.
- Proxying a name the Worker never registered fails at **runtime** ("activity not registered"), not at compile time. Keep the factory the single source so the registered set and the proxied `Activities` type can't drift.

## Durability, identity, replay
- **`workflowId`** is the workflow's identity + dedup key. The orchestrator uses `workflowId = generationId`, so a duplicate `create` for the same generation is a no-op under Temporal's id-reuse policy.
- Temporal persists an **event history** per workflow; on worker restart it **replays** the history to rebuild in-memory state, then resumes. This is the durable-execution guarantee — a crash or deploy mid-run doesn't lose work.
- **Mid-job upgrade**: in-flight workflows finish on the code version they started on; new code affects only new tasks. A **replay test** (re-running new workflow code against recorded histories) is the CI guard that this still holds — see [[Temporal-Workflow-Activity-Boundary]] §1.

## Signals & child workflows
- **Signals** deliver external input to a *running* workflow (the workflow awaits `workflow.condition(...)`). The map-forge parent uses a signal for the clarify→approve wizard instead of the worker's resume-via-DB-state.
- **Child workflows** (`executeChild`) are how the orchestrator fans out — parent → per-level → per-room → per-asset — each independently durable, replacing the worker's manual `withConcurrency`.

## Where it runs
- **Local dev** — the Temporal server + Web UI run in Docker Compose (`docker compose up temporal temporal-ui`; gRPC `localhost:7233`, UI `localhost:8233`).
- **Prod** — **Temporal Cloud** (managed namespace, mTLS); the orchestrator deploys as an always-on Worker polling Temporal Cloud. See `.ai/plans/2026-06-09-adk-temporal-migration/` (Phase 4).
