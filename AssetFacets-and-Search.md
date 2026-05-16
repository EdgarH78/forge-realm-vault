---
Summary: The metadata + retrieval surface for [[Asset]]. `Facets` is a JSONB record with cross-category fields (theme, art_style, material, color_palette, tags, placement, etc.). Postgres maintains a weighted `tsvector` derived from name/description/tags/type via trigger; trigram similarity on name catches typos; JSONB containment + GIN indexes filter on facet equality and tag intersection. `IAssetRepository.search(SearchQuery)` is the single retrieval entrypoint and the only place these mechanisms compose.
Tags: #asset #facets #search #postgres #full-text #atlasforge
---

# AssetFacets-and-Search

## The Facets record

`Facets` is a typed JSONB document declared in `packages/schema/src/domain/Asset.ts`:

```ts
interface Facets {
  type?: string;
  theme?: Theme;                              // Fantasy, Medieval, Sci-Fi, ...
  art_style?: ArtStyle;                       // Realistic, Painterly, Pixel, ...
  material?: string[];                        // ['wood', 'iron']
  color_palette?: string[];                   // ['#a83232', '#1a1a1a']
  tags?: string[];                            // generated (see below)
  size?: SizeCategory;                        // xs|s|m|l|xl|h|g
  wall_type?: 'wooden'|'stone'|'brick'|'magic'|'natural';
  door_type?: 'wooden'|'portcullis'|'stone';
  tileable_type?: 'fence'|'hedge'|'cliff'|'cave'|'natural'|'crop';
  placement?: {
    facing_direction?: 'bottom'|'top'|'left'|'right';
    rotation_support?: 'free'|'cardinal'|'none'|'flip_only';
    wall_attached?: boolean;
  };
}
```

Typed-but-JSONB: adding a new field is non-breaking at the schema level. The structural type lets TypeScript callers narrow safely; the JSONB storage lets Postgres index and query without ALTER TABLE.

`facets.tags` is **derived** — migration 023 backfills `tags` as a lowercased, deduplicated union of every scalar facet field (theme, art_style, type, wall_type, door_type, tileable_type) plus every element of `material[]` and `color_palette[]`. New rows get the same treatment because the `search_text` trigger runs on every INSERT/UPDATE of name/description/facets.

## The search_text tsvector

Migration 004 declares an `update_asset_search_text()` trigger function. On every INSERT/UPDATE of `name | description | facets`, Postgres rebuilds the `search_text tsvector` column from a weighted concatenation:

```
search_text :=
  setweight(to_tsvector('english', name),                'A')   -- highest
| setweight(to_tsvector('english', description),         'B')
| setweight(to_tsvector('english', facets.tags joined),  'C')
| setweight(to_tsvector('english', facets.type),         'C')
```

The weight letters drive `ts_rank_cd`. The trigger fires `BEFORE INSERT OR UPDATE OF name, description, facets` — no application code ever sets `search_text` directly; it's a computed column maintained at write time.

## Indexes (migration 003)

| Index | Type | Purpose |
|---|---|---|
| `idx_assets_search` | GIN(`search_text`) | Full-text retrieval |
| `idx_assets_facets` | GIN(`facets jsonb_path_ops`) | Containment-only — `@>`, `?`, `?|`, `?&` |
| `idx_assets_name_trgm` | GIN(`name gin_trgm_ops`) via `pg_trgm` | Fuzzy / typo-tolerant |
| `idx_assets_category`, `idx_assets_owner`, `idx_assets_visibility`, `idx_assets_pack` | btree | Filter columns |
| `idx_assets_not_deleted` | partial: `WHERE deleted_at IS NULL` | Soft-delete skip |
| `idx_assets_hash` | btree(`hash_sha256`) | Duplicate detection |

`pg_trgm` is enabled at the database level. The trigram index is the difference between "did you mean 'barrell'?" working and not.

## SearchQuery shape

```ts
interface SearchQuery {
  query?: string;                  // free-text
  category?: AssetCategory[];      // OR-filter
  facets?: Partial<Facets>;        // AND-filter via JSONB
  perspectiveClasses?: PerspectiveClass[];
  ownerScope?: 'all'|'mine'|'global'|'entitled';
  userId?: string;                 // required when ownerScope='mine'
  packIds?: string[];
  paletteId?: string;              // restricts to palette membership
  status?: AssetStatus[];          // default ['Active']
  page: number; pageSize: number;
  sort?: { by: 'relevance'|'created'|'name', order: 'asc'|'desc' };
}
```

## search() implementation

`AssetRepositoryPostgres.search(SearchQuery)` builds the WHERE clauses incrementally:

1. `deleted_at IS NULL` and `status = ANY($n)` are always applied (status defaults to `['Active']`).
2. `category` filter — if absent, the query auto-adds `category != 'ConceptArt'` so hero images stay out of library results.
3. `ownerScope` — `'mine'` adds `owner_user_id = $userId`; `'global'` adds `visibility = 'Global'`.
4. `paletteId` — subquery: `id IN (SELECT asset_id FROM palette_items WHERE palette_id = $n)`.
5. **Facet filters** use JSONB operators: scalars via `facets->>'key' = $value`; tag arrays via `facets->'tags' ?| $tagArray` (any-match). All facet filters AND together.
6. **Text search** combines full-text with trigram:
   ```sql
   ( search_text @@ plainto_tsquery('english', $q)
     OR similarity(name, $q) > 0.3 )
   ```
7. **Relevance ordering** when `sort.by === 'relevance'`:
   ```sql
   ORDER BY (ts_rank_cd(search_text, plainto_tsquery('english', $q)) * 2.0
             + similarity(name, $q)) DESC, created_at DESC
   ```

The count query runs in parallel with the page query (`Promise.all`). `SearchResult<T>` carries `items`, `total`, `page`, `pageSize`, and an optional `facetCounts: { facetKey: { value: count } }` for faceted-pivot UIs.

## top_down_compatible (D-18 / migration 029)

A separate column, not a facet — boolean default TRUE, partial index on the FALSE branch only. The migration's heuristic backfill flips FALSE for name matches against `'%portrait%'`, `'%painting%'`, `'%picture frame%'`, `'%wall mural%'`, `'%ceiling mural%'`, `'%tapestry%'`. `findIncompatibleNamePatterns()` returns the FALSE rows' lowercased names — [[MapForgeAgent]]'s `TopDownReferencePhase` Stage 2b post-filter drops inventory entries whose `unique_name` fuzzy-matches the list.

## Cross-domain consumers

- **[[MapItem-Hierarchy]]** placement reads `facets.placement.{facing_direction, rotation_support, wall_attached}` at render and snap time.
- **[[AgenticImageGenerationPipeline]]** `IIntentAssessor` produces a Facets-shaped record that flows into the gen prompt and back into the persisted Asset row.
- **[[MapForgeAgent]]** TopDownReferencePhase calls `AssetResolver` which calls `search()` to dedupe before generating — same description, same facets, hit the cache.
