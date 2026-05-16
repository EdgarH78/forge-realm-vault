---
Summary: The boundary between in-memory immutable [[Map]] state and its persisted JSON form. Transient items are stripped before save; companion shadows are NOT nested children but rebuilt inline on deserialize from parent fields (Pitfall 5); canvas image bytes are NEVER persisted — only asset id references. Versioned migrations chain `N → N+1` and run automatically on load.
Tags: #serialization #persistence #map #atlasforge
---

# MapSerializer

## Identity

`MapSerializer` (`apps/ui/src/atlasforge/map/serialization/MapSerializer.ts`) is constructed with `{ mapItemRegistry: IMapItemSerializerRegistry, vectorRegistry: IVectorSerializerRegistry, appVersion? }`. The two registries are the type-discriminated dispatch surface — every concrete `IMapItem` kind has one registered serializer, every concrete `IVector` type has another. Registration happens at DI bootstrap so the serializer never needs to branch on `kind` itself.

## serialize(map)

```
serialize(map: Map): SerializedMap
  persisted = map.filterTransientItems()                  // strip every Transient
  levels = persisted.levels.map(serializeLevel)
  metadata = { name, size, renderingMode,
               currentLayerIndex, createdAt, appVersion }
  return { version: SERIALIZED_MAP_VERSION, metadata, levels }
```

Each `serializeLevel` writes `{ id, bottomElevation, topElevation, alias?, layers }`. `belowLevelPreview` is **never** written — it's runtime UI state (see [[Layer-and-Level]]). Each `serializeLayer` writes `{ id, name, visible, locked, items }`. Each item passes through `mapItemRegistry.serialize(item, serializeVector)`, where `serializeVector` is bound to `vectorRegistry.serialize` so every nested vector round-trips through its own dedicated serializer (one per concrete type from [[Vector-Math-Extensions]] — `LineVectorSerializer`, `ArcVectorSerializer`, `PolyPathVectorSerializer`, etc.).

## deserialize(data, { context })

```
deserialize(data, { context }): Promise<Map>
  if needsMigration(data): data = migrateToCurrentVersion(data)
  assert data.version === SERIALIZED_MAP_VERSION
  levels = await Promise.all(data.levels.map(deserializeLevel))
  return new Map(metadata.size, metadata.renderingMode,
                 levels[0].layers, metadata.currentLayerIndex,
                 metadata.name, undefined, undefined, levels)
```

Per-level / per-layer deserialization runs in `Promise.all` for I/O concurrency — items often need async image loads. `context.deserializeMapItem` and `context.deserializeVector` are the dispatch primitives the per-kind serializers call back into.

## Versioned migrations

`serialization/migrations/index.ts` maintains a registry of `N → N+1` `MigrationFn`s. `migrateToCurrentVersion` walks from the data's declared version up to `SERIALIZED_MAP_VERSION`, throwing if any step is unregistered. Two migrations ship today:

- **v1 → v2** — remap a 6-layer stack to the 10-layer scheme; introduces `TerrainBackground`, `Objects`, `Shadows`, `Misc`.
- **v2 → v3** — wrap a flat `layers` array into a single default `Level` with elevation 0..10.

New persisted shape changes append a `v(N) → v(N+1)` function; older saves migrate forward transparently.

## DeserializationContext — image bytes never persist

`DeserializationContext` (from `@atlasforge/schema`) exposes `deserializeVector`, `deserializeMapItem`, and `loadCanvasImage(assetId): Promise<CanvasImage | null>`. The serializer writes `CanvasImage` references as `{ assetId, widthPx, heightPx, contentType }` — **the base64 image data is never persisted in the map**. `loadCanvasImage` is the indirection that lets the UI's `AtlasImageManager` and the worker's headless equivalent resolve assets through their respective storage backends.

## Pitfall 5 — companion shadows are rebuilt, not nested

Walls and `MapObject` items each carry a derived `shadow: IShadow | null` companion (constructed by `MapItemFactory` from the parent's shadow params). If we serialized that companion as a *child item*, every round trip would create a duplicate (the parent writes it once, the layer's items list would write it again). Instead:

- **`MapObjectSerializer`** and **`WallSerializer`** persist only the parent's shadow *fields* (`shadows`, `shadowIntensity`, `shadowOffset`, etc.).
- On deserialize, the parent's serializer **rebuilds the companion `Shadow` inline** from those fields, applying backfill defaults for older saves and clamping values to safe ranges (T-05-10 STRIDE mitigation against tampered payloads).

This is the "Pitfall 5" pattern referenced throughout the serialization layer.

## collectAssetIds(map)

`collectAssetIds(map)` walks the map tree and returns every `assetId` referenced by a `CanvasImage`. Preload uses this to fetch image bytes before deserialization runs, so by the time `loadCanvasImage(assetId)` is called the byte cache is already warm.

## The persistence boundary in one sentence

Anything that must survive save and reload lives in the [[Map]] (vectors, kinds, shadow parameters, layer state, level structure). Anything that lives only in PixiJS display objects, the cache hierarchy, the controller stack, or `belowLevelPreview` does not — and `filterTransientItems` enforces it at the boundary.
