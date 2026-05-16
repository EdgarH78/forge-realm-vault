# Architecture: Decoupling

> Tags: #architecture #decoupling #atlasforge

We hit a lot of moving parts in AtlasForge — three deploy targets (UI, API, worker), four storage backends, multiple LLM providers, a Postgres pair, an SSE stream, a render service, and a beta-grade asset pipeline. The system stays coherent because *no two of these know about each other directly*. Every cross-cut is mediated by one of the mechanisms below. When you reach for a shortcut that couples two of them, you are eroding the architecture, not optimizing it.

## 1. Dependency Injection

`AtlasDIContainer` is the universal lookup surface. Bootstrap registers singletons by string key (`vector-runtime-policy`, `vector-factory`, `map-item-factory`, `snap-service`, `ui-style`, etc.); consumers call `di.resolve<T>(key)` or a typed helper (`resolveVectorRuntimePolicy(di)`). Startup-only assertions (`assertVectorRuntimeRegistered`) surface missing wiring at boot, not under load. There are no default fallbacks — a missing registration is a deploy bug, not a runtime degradation. DI is what lets the headless renderer, the worker, and the main editor each construct the same domain types against environment-appropriate adapters.

## 2. Interface-First Contracts

Every cross-package call goes through a TypeScript interface declared *before* its implementation. The catalog: `IVector`, `IMapItem`, `ILayer`, `ILevel`, `ILayerPolicy` ([[Map]] + [[MapItem-Hierarchy]] + [[Layer-and-Level]]), `AtlasCanvas` + `ICacheManager` + `ILevelCacheManager` ([[AtlasForgePixiCanvas]]), `IPromptRefiner` / `IIntentAssessor` / `IImageEvaluator` / `IRenderClient` ([[AgenticImageGenerationPipeline]]), `IStorageService`, `IAssetRepository`, `IMapForgeRepository`. This is what lets `CapturingRenderClient` slot in for the Map Forge review tool with zero changes to [[MapForgeAgent]] — same interface, different implementation, swapped at the DI seam.

## 3. Repository Pattern (and Storage Abstraction)

Domain code never imports `pg`, `@google-cloud/storage`, or `@aws-sdk/client-s3`. It depends on `IAssetRepository`, `IMapRepository`, `IMapForgeRepository`, `IStorageService`. Concrete implementations (`AssetRepositoryPostgres`, `StorageServiceGCS`, `StorageServiceMinIO`, `StorageServiceS3`) live in the `infrastructure/` folder of each app and are wired at DI registration through a factory (`createStorageService()`) that reads env vars. Swapping MinIO for GCS is a config change, not a code change.

## 4. Pub/Sub Event-Driven Architecture

[[PhaseOrchestrator]] publishes lifecycle events to a Google Cloud Pub/Sub topic; the API subscribes and re-emits them as SSE to connected UIs. The producer doesn't know about consumers — review tool, future analytics, retention jobs all subscribe independently. The same model decouples asset ingestion (`asset-ingest` topic), the agent request queue, and image-gen jobs. Every event carries `{source, traceId, generationId, timestamp, ...payload}` — durable in `generation_events`, ephemeral in the SSE channel. The DB write is the *truth*; Pub/Sub is the *liveness signal*.

## 5. Artifact-Based Pipeline Handoff

Phases in the map-generation pipeline do not call each other in-memory. Each phase writes its output as an `artifact` (`artifactRepo.saveArtifact(...)`) with `parentArtifactIds` linking back through the lineage; the next phase loads the latest artifact of the expected type (`planning-contract`, `approved-plan`, `dungeon-ir`, `map-document`, etc.). This makes runs **resumable** (a failed run picks up from the last good artifact), **inspectable** (the review tool reconstructs any phase's state by replaying artifacts), and **decoupled** (no shared in-memory state graph between phases). The exception is the documented untyped `scratchpad` on `PhaseContext` — a deliberate escape hatch for cross-phase hints that must not survive a process restart.

## 6. Schema Mirror Package

`packages/schema` mirrors the UI's vector and map-item types so `apps/worker`, `apps/api`, and `packages/agents` can reason about geometry and authored state without depending on `apps/ui`. Without this, the worker would import from the UI bundle and the monorepo would collapse into one giant cycle. The cost is keeping two type definitions in sync; the benefit is that the worker can run headless in a Docker image one-tenth the size of the UI's.

## 7. Prompt Externalization

LLM prompts live in versioned YAML under `packages/prompt-store`, loaded by name through `PromptStore.getLatestPrompt('map-forge-planner-clarify')`. Code never hardcodes prompt text; it references prompts by stable name and consumes the structured `LLMPromptResult` (JSON-mode results validated by AJV at the prompt-store boundary). This decouples prompt engineering from code review and lets the prompt-store package ship independently. The one operational constraint: prompt YAML changes require `docker compose build worker` to land in the worker image.

## 8. Factories — see [[Architecture-Determinism]]

`VectorFactory` and `MapItemFactory` *look* like decoupling tools, but their primary job is enforcing the immutability and protected-constructor lockdown documented in [[Architecture-Determinism]]. They're the only legal construction surface for vectors and map items, and the CI vector-guard script enforces it. Treat them as determinism scaffolding that happens to provide a swap seam, not as a decoupling pattern on their own.
