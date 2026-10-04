---
Summary: The core immutable domain entity in `packages/schema/src/domain/Asset.ts`. Carries identity (`id`, `versionRootId`, `version`), classification (`category`, `facets`, `categoryData`), ownership (`visibility`, `ownerUserId`, `packId`), file linkage (`assetFileSha256` + optional embedded `AssetFile`), lifecycle status, and non-destructive transforms. Every state change returns a new instance via `with…`-style builders. The single source of truth for asset metadata; the file bytes live elsewhere (see [[AssetStorage]]).
Tags: #asset #domain-entity #immutability #atlasforge
---

# Asset

## Identity & shape

`Asset` is an immutable class with ~30 readonly fields. Identity is the tuple `(id, versionRootId, version)` — see [[AssetVersioning-and-Relationships]] for the chain semantics. Classification is `category: AssetCategory` plus a flexible `facets: Facets` JSONB record (see [[AssetFacets-and-Search]]). File linkage is `assetFileSha256: string` referencing an `AssetFile` row, with an optional embedded `file?: AssetFile` populated by JOINs. Non-destructive transforms — `scaleX`, `scaleY`, `flipX`, `flipY` — defer their effects to render time; the underlying file bytes never change.

The class has no setters. Every mutator returns a new `Asset` with the same `id` (or, for `createNewVersion`, a fresh `id` + same `versionRootId`).

## The category enum

```
'Objects' | 'Walls' | 'TileableObjects' | 'Doors' | 'Windows' | 'FloorTiles'
'Terrain' | 'Material' | 'Brushes' | 'Roofs' | 'Stairs' | 'Roads'
'Decals' | 'LightsFX' | 'Water' | 'Icons' | 'ConceptArt'
```

`'ConceptArt'` and `'Intermediate'` are the map-forge **intermediate-image** categories — generation artifacts, not library assets. `ConceptArt` = per-level hero images; `Intermediate` = top-down room references + render snapshots. Both are **default-excluded from library search** (callers must opt in by passing the category explicitly to the repository search method — exclusion is keyed on **category**, not `mapId`), and `Intermediate` is pruned by retention. `mapId` is **non-null once the image is bound to a committed map** (the worker's `MapForgeAgent` path mints with `mapId` set), but is **`null` for pre-commit captures** — the ADK+Temporal orchestrator mints these before the `maps` row exists, so it binds `mapId` at map-commit time (migration Phase 2f), not at mint. Treat `mapId` as nullable for ConceptArt/Intermediate; never assume it is set.

## Status & lifecycle

`AssetStatus = 'Active' | 'Archived' | 'Drafted' | 'Rejected'`. The four status-builder methods are no-op-when-already-there:

- `archive()` — set status `'Archived'`, bump `updatedAt`.
- `activate()` — set status `'Active'`.
- `draft()` — set status `'Drafted'`.
- `markAsDeleted()` — set `deletedAt` to now; soft delete. Never visible afterwards.

Newly-uploaded assets default to `'Active'`; AI-generated assets default to `'Drafted'` until the user accepts. Phase 8 added `'Rejected'` for hard-rejected candidates from [[AgenticImageGenerationPipeline]] — these are persisted with a `rejected_candidate_for` relationship to the accepted asset so the Map Forge review tool can reconstruct every attempt.

## Visibility & authorization

`Visibility = 'Global' | 'User' | 'PackGranted'`. The authorization rule:

```ts
isVisibleTo(userId, entitledPackIds = [], allowedStatuses = ['Active']):
  if (deletedAt !== null)                                    return false
  if (!allowedStatuses.includes(status))                     return false
  if (visibility === 'Global')                               return true
  if (ownerUserId === userId)                                return true
  if (visibility === 'PackGranted' && packId &&
      entitledPackIds.includes(packId))                      return true
  return false
```

`canBeEditedBy(userId) = ownerUserId === userId`. `canBeDeletedBy(userId) = ownerUserId === userId` (Global-asset admin path is not in MVP). The repository's `search()` (see [[AssetFacets-and-Search]]) enforces these rules in SQL via `ownerScope` and status filters; this class-level method is the per-row check the routes also apply on direct fetches.

## Non-file builders

- `withFile(file?: AssetFile): Asset` — swap the embedded file reference. Used after the ingestion pipeline JOINs file data in (see [[AssetStorage]]).
- `createNewVersion(newData, versionNotes?): Asset` — see [[AssetVersioning-and-Relationships]] for the full semantics; the short version is a new `id`, same `versionRootId`, incremented `version`, `parentVersionId = this.id`, merged facets + categoryData, and **file-related fields must come from `newData` if the file changed** (no auto-detection).
- `getParentAssets(relRepo, assetRepo)`, `getChildAssets(relRepo, assetRepo)`, `decompose(relRepo, assetRepo)` — lineage walks; relationships point at specific version UUIDs, not roots, so historical reproducibility is preserved. See [[AssetVersioning-and-Relationships]].

## Derived getters

- `getUrls(): { original, preview, thumb }` — prefers embedded `file`, falls back to deprecated `urls`. Throws if neither present.
- `getDimensions(): { widthPx, heightPx }` — same fall-through.
- `getContentType()` — `'image/png'`, `'image/webp'`, etc.
- `getDisplaySize(): "{w}×{h}px"`, `getWorldSize(): "{worldW}×{worldH}" | null`.
- `isAIGenerated(): boolean` — `generationPrompt !== undefined`.

## Where Asset shows up

- **[[MapItem-Hierarchy]]**: `ISemanticItem` tags a vector with an `asset: Asset` reference; placement of objects/walls uses `facets.placement` from this entity.
- **[[MapForgeAgent]]**: the worker hero fan-out writes one Asset row per level with `category = 'ConceptArt'` and `mapId` set; the references/render-snapshots are `Intermediate`. The ADK+Temporal orchestrator mints the same categories via a shared `IntermediateAssetPersister` but with `mapId = null` until map-commit (Phase 2f).
- **[[AgenticImageGenerationPipeline]]**: every accepted generation lands as a new `Asset` row (status `'Active'` if accepted as primary, `'Rejected'` if persisted as a candidate fallback).
- **AssetManagerModule** (system_module): the UI grid renders rows of `Asset` straight from `search()`; per-asset inspectors use the builders above.

Bytes never live on the `Asset`. The class is a pure record of metadata, identity, and policy.
