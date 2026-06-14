---
Summary: The background processor. No HTTP surface — every operation arrives as a [[PubSub-Topology]] message. Subscribes to five request topics and dispatches each to its handler: `AssetIngestionService` for new uploads, `ImageGenerationServiceV2` for agentic gen, [[MapForgeAgent]] for map creation, `AssetReprocessHandler` for thumbnail rebuilds, `MapExportProcessor` for dd2vtt exports. Also owns the Patreon-driven credit **grants** ([[CreditSystem]]'s `CreditSyncService` on the tier-change topic) and a tier-reconciliation cron. (Credit **expiry** moved to a durable Temporal Schedule in [[Orchestrator-Service]].) Sync clients reach out to [[ImageService]] and [[RenderService]] over HTTP.
Tags: #service #worker #pubsub #background #atlasforge
---

# Worker-Service

## Stack & bootstrap

`apps/worker/` is TypeScript 5.5 + Node 20.20-slim. `apps/worker/src/index.ts`:

1. Load dotenv.
2. Configure Pub/Sub (emulator vs prod, same logic as [[API-Service]]).
3. Open Postgres pools — same `atlasforge` pool config but smaller (3 conn dev / 10 prod for agents DB).
4. Wire prompt-store from disk (`PROMPTS_PATH` env, default `/app/packages/prompt-store/prompts`).
5. Construct handlers — `AssetIngestionService`, `ImageGenerationServiceV2`, `MapForgeAgent`, `AssetReprocessHandler`, `MapExportProcessor`.
6. Start all subscriptions. **Never opens an HTTP port.**
7. Start cron jobs.

## Subscriptions

| Subscription | Handler | Purpose |
|---|---|---|
| `atlas-asset-ingest-requests-sub` | `AssetIngestionService.processIngest` | Hash → dedup → sharp process → upsert `asset_files` + create `assets` row (see [[AssetStorage]]) |
| `atlas-image-gen-requests-sub` | `ImageGenerationServiceV2.run` | Drives [[AgenticImageGenerationPipeline]], persists accepted asset via `AssetPersister` |
| `atlas-agent-requests-sub` | [[MapForgeAgent]].`handleCreate` / `handleMessage` / `handleApprove` (dispatched on message `type`) | Map generation lifecycle; invokes [[PhaseOrchestrator]] |
| `atlas-asset-reprocess-requests-sub` | `AssetReprocessHandler` | Regenerate previews/thumbs against an existing `asset_file_sha256` |
| `atlas-map-export-requests-sub` | `MapExportProcessor` | dd2vtt assembly via `Dd2vttAssembler` |

Each subscription's handler always acks — success or error. Errors are published as events on the matching `*-events` topic so the [[API-Service]] can relay them as SSE; Pub/Sub redelivery is reserved for outright handler crash. Idempotency is the handler's job (the ingestion service re-dedupes by sha256; map-forge resumes from the latest artifact).

## Sync HTTP clients

The worker reaches out over HTTP to two stateless services:

- `IMAGE_SERVICE_URL` → [[ImageService]] — via `ImageServiceProcessingClient` for `POST /api/process` and via `DirectGeminiImageGenerator` for the multimodal Gemini path. Post-Phase-8 (260430-lx3), generation is direct-to-Gemini from the worker; image-service's `/api/generate` is legacy.
- `RENDER_SERVICE_URL` → [[RenderService]] — `RenderClient` mints a fresh HS256 JWT per call with `iss=atlasforge-render`, `scope=assets:read`, short TTL. Used by `SelfEvalAndFixPhase` to capture per-room PNGs for the self-evaluator. The `IRenderClient` seam lets Plan 13-07's `CapturingRenderClient` slot in for the Map Forge review tool with zero source changes to [[MapForgeAgent]].

## JWT in message envelopes

The [[API-Service]] publishes Pub/Sub messages with the originating user's JWT embedded in the envelope. `apps/worker/src/auth.ts`'s `verifyToken(token)` runs HS256 + `iss=forgerealm-auth` validation on every consumed message before the handler runs. This is defense-in-depth: in the single-publisher model only the API publishes, but the worker still verifies in case a future producer (cron job, admin tool) joins.

## Cron jobs

- `startReconciliationJob` — every 6h reconciles Patreon tier state against `atlasforge.user_tiers` via `CreditSyncService` (the fallback when a tier-change Pub/Sub event is missed). Per-row isolated: one poison auth-DB row logs and is skipped, not stalling the batch. Drives entitlement changes on the API side.

A simple `setInterval` loop under a `pg_advisory_lock` (multi-instance safe), sharing the message handlers' Postgres pool.

> **Credit expiry is NOT here anymore.** The old `startExpiryCronJob` (`setInterval` + advisory lock, burning expired cohorts) was removed in the Formance cutover — expiry is now a daily **Temporal Schedule** (`expireCohortsWorkflow`) in [[Orchestrator-Service]], and balances live in the Formance ledger, not a Postgres `credits` table. See [[CreditSystem]].

## Cross-package imports

The worker imports Postgres repository classes from `../../api/src/infrastructure/AssetRepositoryPostgres`, `AssetFileRepositoryPostgres`, `MapForgeRepositoryPostgres`, etc. This is a deliberate code-sharing seam — the `@atlasforge/schema` package defines the *interfaces* (`IAssetRepository`, etc.); the *Postgres implementations* are shared directly between the API and worker rather than duplicated or extracted into a sixth `apps/db/` package. See [[Architecture-Decoupling]] §3 for the repository-pattern rule.

## Deployment topology

Cloud Run service, scaled on Pub/Sub message throughput rather than HTTP RPS. `concurrency` per instance defaults to 4 (CPU-bound bursts during sharp processing and LLM streaming). No public ingress — the worker has no HTTP listener. Inbound is exclusively Pub/Sub; outbound is to GCS / Postgres / Gemini API / [[ImageService]] / [[RenderService]].
