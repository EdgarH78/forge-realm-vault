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