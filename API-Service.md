---
Summary: The Fastify HTTP gateway on port 3001. Hosts asset/map/credits/map-forge/image-gen/export routes, validates HS256 JWTs (both `forgerealm-auth` user-session tokens and scope-bounded `atlasforge-render` service tokens), subscribes to four event topics on [[PubSub-Topology]] to relay them as SSE, and owns the Postgres connection pools for both the `atlasforge` and `atlasforge_agents` databases.
Tags: #service #api #fastify #sse #atlasforge
---

# API-Service

## Stack & bootstrap

`apps/api/` is Fastify 5 + TypeScript 5.5. Construction in `apps/api/src/index.ts`:

1. Load dotenv (`.env.local` → `.env`).
2. Configure Pub/Sub — set `GOOGLE_CLOUD_PROJECT` defaulted to `scryforge`; routing happens automatically once `PUBSUB_EMULATOR_HOST` is read by the client.
3. Open `pg.Pool`s — `atlasforge` (5 conn dev / 20 prod) and `atlasforge_agents` (3 conn dev / 10 prod). `DATABASE_URL` / `AGENT_DATABASE_URL` env vars; `DATABASE_SSL` toggle.
4. Construct repositories (Postgres) and application services (`AssetService`, `AssetRelationshipService`, `PaletteService`, `MapForgeService`, `CreditService`).
5. Register routes + middleware (auth, rate-limit).
6. Start SSE-relay subscriptions for four events topics.
7. `app.listen(PORT, '0.0.0.0')` — default 3001.

## Route surface

| Mount | Surface |
|---|---|
| `/api/assets/*` | upload init + ingest + search + get/delete + media signing + thumb cache + relationships + decompose + versions + promote + stretch sections + sliced render + reprocess (see [[Asset]], [[AssetStorage]], [[AssetFacets-and-Search]]) |
| `/api/palettes/*`, `/api/map-palettes/*` | palette CRUD and per-map palette pinning |
| `/api/maps/*` | persisted map records + versions + labels |
| `/api/imageGeneration/*` | enqueue [[AgenticImageGenerationPipeline]] runs + SSE stream |
| `/api/assetStatus/*` | bulk status changes |
| `/api/map-export/*` | enqueue dd2vtt exports + SSE stream |
| `/api/mapForge/*` | start / resume / approve / reject [[MapForgeAgent]] generations + SSE per `generationId` |
| `/api/credits/*` | balance, ledger, tier sync |
| `/api/media/sign`, `/api/media/wall-thumb` | signed read URLs + a 200-entry LRU thumb cache |

## Layering — non-negotiable

Every route in `apps/api/` follows the same three-layer shape. **No exceptions for "debug-only", "internal", or "small" routes** — the layering is what keeps the service testable, swappable, and consistent.

```
HTTP handler (apps/api/src/routes/*.ts)
   │   parses request, calls one or more application services,
   │   serializes response. Contains zero SQL, zero business logic,
   │   zero direct database access.
   ▼
Application service (apps/api/src/application/*.ts)
   │   orchestrates repositories, applies business rules, owns
   │   transactions. Returns domain types, not raw rows.
   ▼
Repository (apps/api/src/infrastructure/*Postgres.ts)
   │   the only layer that knows SQL. One repository per aggregate
   │   (Asset, Map, Palette, etc.). Implements an interface defined
   │   alongside the domain entity in packages/schema.
```

**Hard rules:**

- **No inline SQL in route handlers.** If a route needs data, the route imports a service, the service uses a repository, the repository runs the query. A route that pastes a `pool.query(...)` is a bug, not a shortcut.
- **No new fields in route handlers.** Domain shape lives in `packages/schema`. If a route needs to return a new shape, define it there.
- **One repository per aggregate.** Cross-aggregate reads (e.g. "failures across assets + generations") go in an application service that calls both repositories, not in a god-repository.
- **Repositories return domain objects, not rows.** `Row → DomainEntity` mapping happens at the repo boundary so services never see snake_case columns.

This applies to anything new — read endpoints, debug endpoints, admin endpoints, internal tooling endpoints. Same rule.

## Consumers call the API through typed clients

Every consumer of the API — [[UI-Service]], [[Worker-Service]], [[RenderService]] (limited), and any internal tool like the local debug MCP server — talks to the API through a **typed client class**, never `fetch<unknown>` with inline JSON.parse.

- A consumer-side `ApiClient` (or `XxxClient`) class exposes one method per endpoint it uses.
- Each method has an explicit return type. Response shapes are declared as `interface` / `type` in the consumer's own source (or, when sharing makes sense, in `packages/client` / `packages/schema`).
- The client wraps auth (JWT minting / refresh), base URL, error mapping. Tools / route handlers / agents call client methods, never `fetch` directly.
- `unknown` and `any` in API response handling are smells. If you find yourself writing `const x = await fetch(...) as unknown`, stop and add the type.

This is what makes consumers refactor-safe when the API surface evolves: a method signature change surfaces as a TypeScript error, not a runtime crash three calls down.

## Auth middleware

`apps/api/src/middleware/auth.ts` decorates every protected route with `request.user: { userId, tier }`. It accepts **two issuer classes** of HS256 JWT signed with `JWT_SECRET_CURRENT`:

- `iss === 'forgerealm-auth'` — user-session tokens. Admitted on every route.
- `iss === 'atlasforge-render'` — service tokens minted by [[Worker-Service]]'s `RenderClient` for the [[RenderService]] to fetch asset bytes. **Admitted ONLY on the four asset-read endpoints** (D-14 / T-11-01); all other routes reject with 403.

30-second `clockTolerance` absorbs cross-container clock skew (Pitfall 6). When a user-session token is within 30 minutes of expiry, the response carries a freshly minted token in the `X-Refreshed-Token` header — the [[UI-Service]] replaces the in-memory token in place.

## SSE relay pattern

The API is the *only* surface SSE clients (browsers, the review tool) talk to. For each of the four events topics on [[PubSub-Topology]] (`atlas-agent-events`, `atlas-asset-ingest-events`, `atlas-image-gen-events`, `atlas-map-export-events`), `index.ts` spins up a subscription that:

1. Receives a message envelope `{ source, traceId, generationId, ...payload, timestamp }`.
2. Looks up matching SSE connections in an in-memory map (`mapForgeSseConnections`, etc.) keyed by `generationId`.
3. Writes one `event: <type>\ndata: <payload-json>\n\n` to each connection.

The API authors **no** agent state. Every event originates at [[Worker-Service]]; the API only forwards. A reconnecting client re-queries `generation_events` (or its sibling tables) to replay missed events, then attaches the live stream — see [[PubSub-Topology]]'s durability rule.

## Databases

- `atlasforge` — `assets`, `asset_files`, `asset_relationships`, `asset_stretch_sections`, `palettes`, `palette_items`, `maps`, `map_versions`, `map_labels`, `credits_*`, etc. Migrations: `apps/api/migrations/*.sql`.
- `atlasforge_agents` — the `agent_runtime` schema: `sessions`, `turns`, `tasks`, `messages`, `tool_calls`, `prompt_calls`, `artifacts`, `checkpoints`, `generations`, `generation_events`. Migrations: `apps/api/migrations/agents/*.sql`.

Both pools shut down cleanly on `SIGINT` — in-flight queries drain, then `pool.end()`.

## Cross-service calls (sync HTTP)

The API talks to [[ImageService]] over HTTP for synchronous asset-reprocess paths (`ImageServiceProcessingClient.process(buffer, mime, options)`). It does **not** call image-service `/api/generate` — generation lives entirely in [[Worker-Service]] post-Phase-8.

## Rate limiting & graceful shutdown

`@fastify/rate-limit` registered globally; per-route overrides on heavy endpoints. `SIGINT` / `SIGTERM` triggers ordered shutdown: stop accepting new HTTP connections → drain in-flight requests (default 30s) → unsubscribe Pub/Sub subscriptions → `pool.end()` on both DBs → process exit.
