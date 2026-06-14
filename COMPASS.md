# Engineering Compass: AtlasForge

## 🎯 Core Philosophy
We build strictly **boring, deterministic, and decoupled** backend architectures. 
Velocity must never be bought by sacrificing system predictability. No runaway automation loops.

## 🧱 Architectural Guardrails
* **Decoupled State Engine:** The visual or rendering canvas (PixiJS) is a pure projection layer. Geometric state logic must live in immutable, interface-driven vector models.
* **Interface-First Engineering:** Before writing any implementation logic, declare the TypeScript interfaces or contracts. 
* **Human Sovereignty:** Claude Code must operate in Plan Mode first. No automated file modification loops or blind commits allowed.

## 🗺️ System Index

### Foundational Rules
* [[Architecture-Determinism]] — Rules for state mutation and logic isolation.
* [[Architecture-Decoupling]] — The seven mechanisms that keep no two subsystems knowing about each other directly.
* [[Type-Discipline]] — Type safety beats convenience: no `null`/`undefined` for domain values, no `unknown` past a deserialization boundary, no `any` anywhere. Applies to writing code, not just reviewing it.

### Subsystem: Canvas & Geometry
* [[PolyPathVector]] — Composite immutable vector; the canonical multi-segment shape.
* [[Vector-Math-Extensions]] — Pure-function kernels under `vectors/geometry/`; total, deterministic, policy-parameterized.
* [[AtlasForgePixiCanvas]] — The single PixiJS adapter; one-way projection layer from immutable state to display tree.

### Subsystem: Map Domain
* [[Map]] — Aggregate root; immutable; orchestrates render and is the boundary that crosses into the canvas.
* [[MapItem-Hierarchy]] — `IMapItem` contract, the three abstract base classes, and the concrete-kind catalog (Wall, Tile, Door, etc.).
* [[Layer-and-Level]] — Container shapes that organize items; layer policy routes new items to their semantic home.
* [[MapSerializer]] — JSON round-trip boundary; transient stripping, versioned migrations, asset-reference-only image persistence.

### Subsystem: Service Architecture
* [[UI-Service]] — Vite/React/PIXI browser client (port 5173); plugin-based modules; SSE consumer.
* [[API-Service]] — Fastify gateway (port 3001); routes, JWT middleware, SSE relay, both Postgres pools.
* [[Worker-Service]] — Pub/Sub-driven background processor; no HTTP surface; cron + sync clients to ImageService and RenderService.
* [[PubSub-Topology]] — Six request/events topic pairs (plus DLQ); request/event duality; durable DB write + ephemeral liveness signal; the canonical drift surface.
* [[ImageService]] — .NET 8 stateless processor (port 5100); chroma-key / SAM / bbox-crop / resize / WebP.
* [[RenderService]] — Headless Chromium (port 5200); single warm Playwright browser; renders SerializedMap → PNG.
* [[Deployment]] — GitHub Actions → Cloud Build pipeline; parallel image builds, ordered Pub/Sub + DB + service rollout; two URL-injection passes wire services together.

### Subsystem: Asset Domain
* [[Asset]] — Core immutable entity; identity, status, visibility, builders, authz rule.
* [[AssetFacets-and-Search]] — JSONB facets, weighted `tsvector` trigger, trigram fuzzy match, GIN indexes; the `search(SearchQuery)` surface.
* [[AssetStorage]] — `AssetFile` content-dedup by SHA-256, sharded GCS paths, 5 MB cap; `IStorageService` abstraction; init-upload → staging → Pub/Sub ingest lifecycle.
* [[AssetVersioning-and-Relationships]] — Version chains, lineage edges, stretch sections; relationships point at specific version UUIDs for reproducibility.

### Subsystem: AtlasUI
* [[AtlasUIElement]] — Framework-neutral discriminated union for every UI primitive; plain data, no React, no PixiJS.
* [[FunctionalComponent-and-Renderer]] — Component runtime: `AtlasForgeFC`, multi-pass renderer, keyed hooks, path-based reconciliation.
* [[AtlasModule-and-AtlasLibrary]] — Plugin architecture: modules with modes, libraries, selection-priority dispatch, `system_modules/` catalog.
* [[AtlasUIElementRenderer]] — The React boundary; the single 1,375-line dispatcher; the only file that knows AtlasUI primitives in React land.

### Subsystem: Agent Pipeline
* [[DeepAgent]] — Generic plan-reflect-execute loop; the open-ended runtime under `PromptAssistantAgent`.
* [[MapForgeAgent]] — Concrete orchestrator for AI map creation; drives the clarify→plan→approve→execute lifecycle.
* [[PhaseOrchestrator]] — Deterministic sequential phase runner; dual-channel events; rigid counterpart to DeepAgent.
* [[AgenticImageGenerationPipeline]] — Inner assess→refine→generate→evaluate retry loop with severity-classified gates.
* [[Test-Capture-Wrappers]] — The `_*` invoke-var convention by which agent call sites stamp room / attempt / promptName metadata onto `rendered.parameters`, where the auditing decorators read it for capture attribution. `_promptName` is system-reserved (non-spoofable).

### Subsystem: Orchestrator (ADK + Temporal)
* [[Temporal]] — The durable workflow runtime: workers poll a task queue and run deterministic workflows + side-effecting activities. Activities are registered via `Worker.create({ activities })` and invoked by name at runtime. Registration/proxy mechanics, signals, child workflows, replay, local-vs-Cloud.
* [[Orchestrator-Service]] — The `apps/orchestrator` Temporal worker replacing the hand-rolled agent pipeline ([[MapForgeAgent]] / [[PhaseOrchestrator]] / [[AgenticImageGenerationPipeline]]) with ADK agents on Temporal. In-progress migration; Pub/Sub stays ingress+egress so the API/UI contract is unchanged.
* [[Temporal-Workflow-Activity-Boundary]] — The load-bearing orchestrator rules: deterministic workflows vs side-effecting activities, coarse activity grain, the closure-DI factory, idempotent side-effects, and the byte-compatible egress envelope.

### Subsystem: Credits & Billing
* [[CreditSystem]] — The credit economy on a self-hosted **Formance OSS ledger** (double-entry, append-only): cohorts as accounts, FIFO spend via ordered Numscript source sets, 3-month expiry burns, idempotency via the ledger `Idempotency-Key`, deterministic billing-cycle-keyed grants, cursor-only pagination, and `GET /api/credits/history` straight off the audit-trail-that-is-the-money. Postgres keeps only config (`action_costs`/`tier_credit_allotments`/`user_tiers`); ledger I/O is Temporal-activities-only.

### Subsystem: Review & Capture Tooling
* [[Review-Service]] — Localhost-only Fastify backend + React SPA for human review of AI output; one source — the `agent_runtime` audit DB — with signed asset images and reviewer tags persisted to `stage_tags` via a narrowly-scoped RW pool.
* [[Review-NodeTree-UI]] — The review frontend: a data-driven LangGraph-style node tree rendering the `agent_runtime` stage/action/artifact tree as-is (no client reconstruction), with lazy artifact bodies, Monaco/image-viewer dispatch, per-node `nodeRef` tagging, and live-run polling.
* [[Review-Capture-Pipeline]] — How Map Forge + Asset Forge runs get captured: the auditing decorators (`AuditingLLM` / `AuditingImageGenerator` / `AuditingRenderClient`) write to the `agent_runtime` tree via `AuditRunContext`; `AssetPersister` links accepted + rejected assets with a P-3 verdict.