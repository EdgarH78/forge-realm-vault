---
Summary: The (now slim) background processor. No HTTP surface — every operation arrives as a [[PubSub-Topology]] message. Subscribes to the remaining request topics: `AssetIngestionService` for new uploads, `AssetReprocessHandler` for preview/thumbnail rebuilds, `MapExportProcessor` for dd2vtt exports. **The agentic image-gen and map-forge handlers moved to [[Orchestrator-Service]]** (image-gen in Phase 1; map-forge at cutover #611, agent code deleted #613) — the worker no longer subscribes to `atlas-image-gen-requests` or `atlas-agent-requests`, and no longer calls [[RenderService]]. Also owns the Patreon-driven credit **grants** ([[CreditSystem]]'s `CreditSyncService` on the tier-change topic), a tier-reconciliation cron, and audit/asset retention jobs. (Credit **expiry** moved to a durable Temporal Schedule in [[Orchestrator-Service]].) Sync client reaches out to [[ImageService]] over HTTP.
Tags: #service #worker #pubsub #background #atlasforge
---

# Worker-Service

## Stack & bootstrap

`apps/worker/` is TypeScript 5.5 + Node 20.20-slim. `apps/worker/src/index.ts`:

1. Load dotenv.
2. Configure Pub/Sub (emulator vs prod, same logic as [[API-Service]]).
3. Open Postgres pools — same `atlasforge` pool config but smaller (3 conn dev / 10 prod for agents DB).
4. Wire prompt-store from disk (`PROMPTS_PATH` env, default `/app/packages/prompt-store/prompts`).
5. Construct handlers — `AssetIngestionService`, `AssetReprocessHandler`, `MapExportProcessor`, `CreditSyncService`. (`ImageGenerationServiceV2` and `MapForgeAgent` were removed — those paths now run in [[Orchestrator-Service]].)
6. Start all subscriptions. **Never opens an HTTP port.**
7. Start cron + retention jobs.

## Subscriptions

| Subscription | Handler | Purpose |
|---|---|---|
| `atlas-asset-ingest-requests-sub` | `AssetIngestionService.processIngest` | Hash → dedup → sharp process → upsert `asset_files` + create `assets` row (see [[AssetStorage]]) |
| `atlas-asset-reprocess-requests-sub` | `AssetReprocessHandler` | Regenerate previews/thumbs against an existing `asset_file_sha256` |
| `atlas-map-export-requests-sub` | `MapExportProcessor` | dd2vtt assembly via `Dd2vttAssembler` |
| `atlas-tier-change-events-sub` | `CreditSyncService` | Patreon tier-change → credit grant ([[CreditSystem]]) |

> **Moved to [[Orchestrator-Service]]:** `atlas-image-gen-requests-sub` (was `ImageGenerationServiceV2` → [[AgenticImageGenerationPipeline]]) and `atlas-agent-requests-sub` (was [[MapForgeAgent]] → [[PhaseOrchestrator]]) are now consumed by the orchestrator's Temporal ingress. Removed from the worker at Phase 1 / the map-forge cutover (#611, #613).

Each subscription's handler always acks — success or error. Errors are published as events on the matching `*-events` topic so the [[API-Service]] can relay them as SSE; Pub/Sub redelivery is reserved for outright handler crash. Idempotency is the handler's job (the ingestion service re-dedupes by sha256).

## Sync HTTP clients

The worker reaches out over HTTP to one stateless service:

- `IMAGE_SERVICE_URL` → [[ImageService]] — via `ImageServiceProcessingClient` for `POST /api/process`, used by `AssetReprocessHandler`'s preview/thumbnail regeneration (a deliberate subset of the old agentic pipeline). The worker's direct-to-Gemini generation path and its [[RenderService]] client (`RenderClient`, formerly used by the deleted `SelfEvalAndFixPhase` to capture per-room PNGs) both moved to [[Orchestrator-Service]] — generation is now the `generateAndProcess` activity, per-room rendering the `renderRoom` activity (mints its own per-call render JWT).

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
