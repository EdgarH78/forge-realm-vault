---
Summary: The AtlasForge Review Service — a localhost-only Fastify backend + React SPA for human review of AI-generated Map Forge and Asset Forge runs. **One source:** the `agent_runtime` audit DB (the stage·action·artifact tree the worker's auditing decorators write). The file-capture path that used to mirror the same wire contract off `apps/worker/test-fixtures/**/__output__/` was retired in PRs R-3 + R-4 (2026-06). The projection is now **tree-shaped**: `projectRunTree` / `projectNodeDetail` ship the audit tree 1:1 (the old flatten-and-reconstruct `projectManifest`/`projectStageDetail` path and the per-asset gallery were deleted in the LangGraph redesign Phase 6, 2026-06). Reviewer tags persist back to `agent_runtime.stage_tags` on a separate, narrowly-scoped RW connection; asset images serve as short-lived signed URLs (GCS in prod, MinIO locally). Still localhost-only, no-auth. The capture side is [[Review-Capture-Pipeline]]; the UI is [[Review-NodeTree-UI]].
Tags: #review #tooling #atlasforge #service-architecture
---

# Review Service

A standalone tool for a human (currently just the author) to walk AI-generated Map Forge runs and Asset Forge (bg-removal) outputs and tag where the model's judgment was right or wrong. It is **not** part of the production mesh — it has no auth and binds to `127.0.0.1`. Lives at `apps/review-service/{backend,frontend}`. Reads production runs read-only from `agent_runtime` via a Cloud SQL proxy operators run locally.

## Why it exists

The agent pipelines ([[MapForgeAgent]], [[AgenticImageGenerationPipeline]]) make hundreds of model judgments per run — readouts, evaluations, fix plans, retry strategies. When a map comes out wrong, you need to attribute the failure: was the readout wrong, the asset generator, the placement, the evaluator's accept/reject call? The review tool lets a reviewer walk the run as a sequence of events and attach a structured judgment (`agree` / `disagree` / `uncertain`) to each model decision, building a labelled corpus of where the pipeline fails.

## One source: the agent_runtime audit DB

Every run — Map Forge and Asset Forge — is captured into `agent_runtime.{stages, actions, action_artifacts}` by the auditing decorators ([[Review-Capture-Pipeline]]). The review service projects that tree **as-is** into the tree-shaped `RunTree` / `NodeDetail` wire contract the frontend consumes (`auditProjection.ts`: `projectRunTree` ships structure + cheap fields, `projectNodeDetail` resolves one node's artifact bodies lazily). The frontend renders it directly with no reconstruction. The earlier flatten path (`projectManifest` → flat `MapForgeManifest.stages[]`, re-bucketed by `promptName`) and the separate asset-gallery projection were deleted in Phase 6.

The Phase-D `REVIEW_DB_ENABLED` flag and the file-capture sibling repos (`RunRepository`, `MapForgeRunRepository`, `MapForgeAssetGallery`, `AssetReviewAggregator`) were removed by PRs R-3 / R-4 / R-5 once the audit-tree path reached parity (R-1 linked accepted+rejected assets via `asset-persist` actions; R-1b extended capture to the Asset Forge / bg-removal-retry generation path; R-2 + later P-3 added the per-asset gallery, verdict (attempt history), final-render snapshot, and v1 raw image).

## Three pools, one direction of write

| Pool | Driver | Permissions | Used by |
|------|--------|-------------|---------|
| `agent_runtime` RO | `agentDbPool` | `default_transaction_read_only=on` | `MapForgeRunRepositoryDb`, signed-image route |
| `atlasforge` RO (assets) | `assetDbPool` | RO | `AssetPathResolver` (signed-URL resolution), `AuditAssetResolver` (label + rejected-parent + v2→v1 raw resolution) |
| `agent_runtime` RW (tags only) | `stageTagsRwPool` | grant: `INSERT/DELETE` on `agent_runtime.stage_tags` only | `StageTagWriter` |

The RW pool is optional at construction. Unset ⇒ tag writes 503; reads stay available. Localhost / no-auth gating is uniform across all three.

## Cross-pool soft refs

`action_artifacts.asset_id` is a soft cross-DB ref — there's no FK across the agent_runtime / atlasforge boundary. `AuditAssetResolver` is the single seam that hops the boundary, exposing three lookups:

- `resolveLabels(assetIds)` — `asset_id → name`. Doubles as the existence check: an id absent from the result was pruned (retention) and the gallery omits it.
- `resolveRejectedParents(childIds)` — rejected child `asset_id` → its accepted parent via `asset_relationships` (relationship_type=`rejected_candidate_for`). The CONCURRENCY-SAFE grouping key for assembling the per-asset attempt history (furnishings persist concurrently within one phase frame, so audit-tree sequence order can't separate one asset's rejects from another's).
- `resolveRawParents(acceptedIds)` — v2 (accepted processed) `asset_id` → v1 raw `asset_id` via `assets.parent_version_id`. Surfaces as `images['final-raw']` so the reviewer sees the pre-bg-removal Gemini output alongside the processed final.

All three follow the same "row absent → silently omit" policy.

## Image signing

The projector emits `<img src="/images/agent-asset/{assetId}">`; that route resolves the object key (`assets.asset_file_sha256` → `asset_files.original_url`) and 302-redirects to a short-lived v4-signed URL. GCS in prod, MinIO direct in local dev (no signing — the `MINIO_PUBLIC_ENDPOINT` URL is the redirect target). TTL is `signedUrlTtlSeconds`, defaulting to 300 s; URLs are re-minted per request.

## Tag model

One `TagEntry` per reviewer assertion, persisted to `agent_runtime.stage_tags`:

```typescript
interface TagEntry {
  judgment: 'agree' | 'disagree' | 'uncertain';
  target: 'image-gen' | 'evaluator' | 'refiner' | 'pipeline-decision'
        | 'planner' | 'concept-style' | 'top-down-readout' | 'validator'
        | 'self-evaluator' | 'fix-planner' | 'render';
  stage: string;        // free-form path, e.g. 'attempt-1.evaluator.failureMode'
  comment: string;
  timestamp: string;    // ISO-8601 — server-authoritative (stage_tags.created_at)
  reviewerId: string;   // hardcoded 'edgar' for the single-reviewer MVP
}
```

`target` enumerates the model decision-makers and grows additively. `VALID_TARGETS` / `VALID_JUDGMENTS` and the `tagValidation.validateTagEntry` predicate single-source the shape check; the frontend `types.ts` mirrors the union (a known drift surface — keep them in lock-step).

The frontend matches tags to wire stages by `tag.stage === stage.name`, but one coarse audit stage fans out into several wire stages — the name can't be reconstructed from `stage_id`. It's therefore stored verbatim in `stage_tags.stage_label`; `stage_id` carries the run's root stage for FK/cascade.

## Identifier model

- `runId` = `generation_id` (a UUID) — groups one user-facing generation.
- `runDir` = the root `stage_id` (also a UUID) — identifies one tree under that generation. A generation has exactly one root stage today, so a run lists exactly one `runDir`.

The DB indexer (`listRuns`) reads only root stages (`parent_stage_id IS NULL`); a run's `has_review` flag is a subquery against `stage_tags` joined through `stages` and `actions`.

## Server shape

`backend/src/index.ts` constructs the three pools, the `AuditAssetResolver` (feeds the tree projector's asset descriptors), the `MapForgeRunRepositoryDb`, the signed-image router, and the optional `StageTagWriter`. Routes register in two modules: `mapForgeRuns.ts` (run-dir listing + `/tree` + `/nodes/:nodeId` + tag CRUD) and `agentAssetImages.ts` (signed redirect); `runs.ts` serves the cross-pipeline `/api/runs` index. Binds `127.0.0.1:3002` by default; Docker overrides host via `REVIEW_SERVICE_HOST`. **No auth by design** — localhost-only, single reviewer.

UUID validation is uniform and load-bearing: `UUID_RE` gates every `runId` / `runDir` / `fixtureId` before the repo touches the DB. The seam returns null (→ 404) on unknown ids rather than throwing.

## Boundaries

- Read the codebase for exact route signatures and response shapes — `backend/src/types.ts` is the authoritative wire contract.
- The UI structure (node tree, node-detail tabs, artifact rendering, per-node tagging) is its own atom: [[Review-NodeTree-UI]].
- How the tree gets *written* by the auditing decorators is [[Review-Capture-Pipeline]].
- Production hardening (Google IAM OAuth, multi-reviewer, deploy posture) is deferred and explicitly out of scope today.
