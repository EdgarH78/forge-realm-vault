---
Summary: Two parallel histories on [[Asset]]. **Versioning** chains assets in time — same `versionRootId`, increasing `version`, `parentVersionId` link, atomic trigger-set root id. **Relationships** chain assets in lineage — `extracted_from`, `assembled_from`, `rejected_candidate_for`, with sequence ordering and metadata. Relationships and version-edges both point at *specific version UUIDs*, never roots, so historical decomposition is reproducible. `AssetStretchSection` adds a third orthogonal axis: per-asset horizontal segmentation into locked vs. stretching bands for windows and doors.
Tags: #asset #versioning #relationships #lineage #atlasforge
---

# AssetVersioning-and-Relationships

## Versioning columns

Migration `015_add_asset_versioning.sql` adds four columns to `atlasforge.assets`:

```
version_root_id    UUID NOT NULL          -- stable group identifier
version            INT  NOT NULL DEFAULT 1
parent_version_id  UUID  REFERENCES assets(id) ON DELETE SET NULL  -- NULL for v1
version_notes      TEXT                   -- free-form
```

`version_root_id` is the **stable identity** of the version family. For a v1 asset, `version_root_id = id`. For vN (N>1), `version_root_id = the root's id`. Rename-safe: renaming the asset doesn't change the root id, so all versions stay linked.

## The auto-root trigger

`trigger_set_version_root_id` (migration 015) fires `BEFORE INSERT` and, when `version = 1 AND parent_version_id IS NULL`, sets `version_root_id := NEW.id`. This is what makes "create a v1 asset" atomic — no separate UPDATE, no application code to coordinate. New code paths that need version chains can omit the versioning fields entirely and rely on the trigger; chain extensions (v2, v3) must supply `version_root_id`, `version`, `parent_version_id` explicitly.

## Indexes

- **`idx_assets_version_root_version`** — `UNIQUE(version_root_id, version) WHERE deleted_at IS NULL`. One row per version number per root; soft-deleted versions don't block the slot.
- **`idx_assets_version_root_latest`** — `(version_root_id, version DESC)`. The "give me the latest version" query is a 1-row index lookup.
- **`idx_assets_parent_version`** — `(parent_version_id)`. Reverse-walk by parent.

## Asset.createNewVersion

The method on [[Asset]] that produces the next version:

```
createNewVersion(newData, versionNotes?):
  newId = randomUUID()
  versionRootId = this.versionRootId || this.id   // seed root if this is v1
  return new Asset(
    newId,                                       // fresh id
    merged fields from newData,
    versionRootId,                               // stable across versions
    this.version + 1,                            // bumped
    this.id,                                     // parentVersionId
    versionNotes,
    ...
  )
```

Three load-bearing rules:

1. **`assetFileSha256` and `file` must come from `newData` if the file changed** — there is no auto-detection. Caller is responsible.
2. **`ownerUserId`, `creatorId`, and `mapId` persist across versions.** A new version of a concept-hero asset stays associated with the same map.
3. **`facets` and `categoryData` are merged**, not replaced — `{ ...this.facets, ...newData.facets }`. Drop a field by setting it to `undefined` in `newData`.

`findByVersionRoot(rootId)` returns every version of a root ordered by `version`. `Asset.getVersionHistory(assetRepo)` is the convenience wrapper.

## AssetRelationship

Migration `014_create_asset_relationships.sql`:

```
asset_relationships (
  id UUID PRIMARY KEY,
  parent_asset_id UUID REFERENCES assets(id) ON DELETE CASCADE,
  child_asset_id  UUID REFERENCES assets(id) ON DELETE CASCADE,
  relationship_type asset_relationship_type,
  sequence_order INT DEFAULT 0,        -- ordering within parents
  metadata JSONB DEFAULT '{}'::jsonb,
  CONSTRAINT no_self_reference CHECK (parent_asset_id != child_asset_id)
)
```

`asset_relationship_type` enum: `'extracted_from' | 'assembled_from' | 'rejected_candidate_for'`. Composite indexes on `(child, type, sequence_order)` and `(parent, type)` cover the two common lookup directions.

## The version-handling rule (load-bearing)

`parent_asset_id` and `child_asset_id` are UUIDs from `assets.id` — they point at a **specific version**, not at a version root. When a parent asset is updated to a new version, its old `id` still exists; the relationship continues to reference the version that was authoritative when the relationship was created. This is what makes "decompose" reproducible.

`Asset.getParentAssets(relRepo, assetRepo)`:

```
relationships = relRepo.findByChildId(this.id)
sort by sequenceOrder
parentIds = relationships.map(r => r.parentAssetId)
return assetRepo.findByIds(parentIds)            // exact versions, by UUID
```

`getChildAssets` is symmetric. `decompose` is an alias for `getParentAssets`. None of these methods follow version chains — they return the exact rows the relationship pointed at.

## Three concrete use cases

- **`extracted_from`** — a wall slice extracted from a material asset. The relationship's `metadata` carries slice parameters (start/end percentages, edge feathering).
- **`assembled_from`** — a kitbashed asset built from N source parts. `sequence_order` is the assembly order; the same parent can appear multiple times (3 rubies in one assembly), each with its own row id.
- **`rejected_candidate_for`** — added Phase 8. Hard-rejected candidates from [[AgenticImageGenerationPipeline]] persist as `status = 'Rejected'` asset rows with this relationship pointing at the accepted asset. The Map Forge review tool reconstructs the per-asset attempt history by walking these edges.

## AssetStretchSection — orthogonal third axis

Migration `020_create_asset_stretch_sections.sql`:

```
asset_stretch_sections (
  id UUID PRIMARY KEY,
  asset_id UUID REFERENCES assets(id) ON DELETE CASCADE,
  section_order INT,
  start_pct FLOAT, end_pct FLOAT,    -- 0.0 .. 1.0 of asset width
  locked BOOLEAN DEFAULT TRUE,
  CONSTRAINT valid_range CHECK (start_pct >= 0 AND end_pct <= 1 AND start_pct < end_pct),
  CONSTRAINT unique_section_order UNIQUE (asset_id, section_order)
)
```

Percentages are resolution-independent — a 50% mark on a 1024px-wide asset and the same 50% mark on its 2048px high-res version refer to the same logical position. `locked = true` means the section maintains aspect ratio during stretch rendering; `locked = false` means it stretches to fill remaining space. Window and door assets use this to pin end caps and stretch the middle. The renderer in [[AtlasForgePixiCanvas]]'s `drawAssetPath` consumes the sections via the asset metadata pipeline; see [[MapItem-Hierarchy]] for which kinds rely on it.

## What this domain does NOT track

Lineage and versioning intentionally don't replace audit logging. There is no edit history per field, no "who changed this when" trail beyond `created_at` / `updated_at`. If you need that level of history, write it as an `assembled_from` chain with version bumps — the granularity is "new immutable artifact", not "tracked field change".
