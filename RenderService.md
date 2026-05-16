---
Summary: The headless-Chromium map-rendering microservice on port 5200. Hosts the [[UI-Service]]'s headless bundle at `/headless/` and exposes `POST /render` that drives a single warm Playwright browser to produce a PNG from a `SerializedMap`. Used by [[MapForgeAgent]]'s `SelfEvalAndFixPhase` to evaluate compiled maps in-loop. JWT is forwarded verbatim to [[API-Service]] for asset fetches inside the page; [[API-Service]] enforces the scope bound.
Tags: #service #render #headless #playwright #atlasforge
---

# RenderService

## Stack & shape

`apps/render-service/` is Fastify 5 + Playwright 1.58 + TypeScript 5.5. Single binary, single process, single warm Chromium. `Dockerfile.render-service` ships Chromium + the UI's `dist/` bundle. Port 5200. `/health` is wired to return 503 *until* `BrowserPool.start()` completes, so the [[Worker-Service]] sees the service as ready only when it can actually render.

## BrowserPool — exactly one warm page

`apps/render-service/src/browser/BrowserPool.ts` owns one `Browser` + one `BrowserContext` + one `Page` for the lifetime of the process. Cloud Run `concurrency = 1` (D-08) — exactly one render is in flight at any moment, so the single-page design fits the deployment.

Launch options (`launchOptions.ts`): `--no-sandbox`, `--disable-gpu`, `--disable-dev-shm-usage`, `--shm-size=2gb` provided externally by docker-compose. Each `render()` call:

1. Installs a `page.route('**', handler)` for outbound asset fetches.
2. `page.goto('/headless/?headless=1')` if the page hasn't loaded the bundle yet (warm-path: skip).
3. `page.evaluate(window.__renderMap, { map, ...dims })` with a 10-second timeout (`RENDER_TIMEOUT_MS`).
4. Extracts the rendered PNG as base64 via `PIXICanvasExtractor`.
5. `page.unroute(...)` in a `finally` so the per-request JWT closure doesn't leak to the next render (W3 fix).

`BrowserNotReadyError` and `RenderTimeoutError` bubble up to the route handler for HTTP error mapping.

## Static server — the UI's headless bundle

`registerStaticUiDist(app)` mounts `apps/ui/dist` at `/headless/`. In dev, the volume is bind-mounted: `./apps/ui/dist:/app/apps/ui/dist:ro` — frontend rebuilds in the [[UI-Service]] container immediately become available to the render service without rebuilding `Dockerfile.render-service`. In prod, `dist/` is baked into the image at build time.

The headless bundle is one of two entry points produced by Vite (`vite.config.ts` inputs `main` and `headless`). It strips chrome, panels, and input controllers, leaving exactly the canvas + the `window.__renderMap(opts)` function — enough to render a map and nothing more. This is the same WorldCanvas / [[AtlasForgePixiCanvas]] that the [[UI-Service]] uses; identical geometry, identical output, by [[Architecture-Determinism]] design.

## POST /render

Two body shapes are accepted (Phase 13.2 Wave 2):

- **Legacy** — body is `SerializedMap`. `computeRenderDimensions(map)` derives the bbox from map geometry.
- **Envelope** — body is `{ map: SerializedMap, renderOpts?: RenderBbox }`. `renderOpts` carries six pre-computed fields `{ worldOriginX, worldOriginY, worldWidth, worldHeight, width, height }` used verbatim, bypassing server-side bbox math. [[MapForgeAgent]]'s `SelfEvalAndFixPhase` always sends the envelope so per-room renders frame exactly the room.

Auth: `Authorization: Bearer <jwt>` required, 401 if missing. **The JWT is not verified here** — it's forwarded verbatim to [[API-Service]] for asset fetches; the API middleware is the verifier (D-13/D-14). This is the documented one-way trust — the render service trusts inbound callers (a tampered token will be rejected when it hits the API), but it never elevates privilege.

Error mapping: `RenderTooLargeError` → 413, `EmptyMapError` → 400, `RenderTimeoutError` → 504.

## Asset-fetch interception

When the headless page's [[AtlasForgePixiCanvas]] resolves `CanvasImage` references, it issues `fetch` calls for `/api/media/sign` or signed URLs that resolve to GCS / MinIO. The route handler installed at start of each render:

1. Sees the request inside the page context.
2. Decides whether to fulfill or continue based on the URL.
3. For backend requests, adds `Authorization: Bearer <jwt>` from the per-render closure.
4. For dev: rewrites `localhost:9000` → `minio:9000` per `RENDER_STORAGE_REWRITE_FROM/TO` env vars so the headless browser inside the container can reach MinIO over the docker network.
5. `route.continue()` with the modified request.

The JWT in the closure is the same token the caller (typically [[Worker-Service]]'s `RenderClient`) provided. Its `iss=atlasforge-render, scope=assets:read` claim is what limits the blast radius — [[API-Service]] admits this token only on the four asset-read endpoints, and rejects everything else with 403.

## Caller — RenderClient

[[Worker-Service]] holds an `IRenderClient` instance (production: `RenderClient`; review tool: `CapturingRenderClient`). Each call mints a fresh HS256 JWT (`iss=atlasforge-render`, `scope=assets:read`, short TTL ~5min), POSTs to `/render`, and receives PNG bytes. The DI seam in [[MapForgeAgent]]'s constructor (`MapForgeAgentOptions.createRenderClient`) lets the Map Forge review tool swap in the capturing wrapper without source changes.

## Graceful shutdown

`SIGINT` / `SIGTERM` triggers:

1. `app.close()` — stop accepting new HTTP connections, drain in-flight requests.
2. `browserPool.stop()` — close the page, context, and browser cleanly so Chromium doesn't leave orphan processes.

Both run in independent `try / catch` blocks (B4 fix) — if `app.close()` hangs, the browser still gets cleaned up; if `browserPool.stop()` errors, the HTTP server already drained.

## What this service is NOT

- **Not stateful.** Same map → same PNG (modulo Chromium font-rendering quirks across versions, which are pinned by `Dockerfile.render-service`).
- **Not a generic browser farm.** One renderer, one purpose: turn a `SerializedMap` into a PNG. No scraping, no PDF generation, no screenshotting arbitrary URLs.
- **Not the only place [[AtlasForgePixiCanvas]] runs.** It runs identical canvas code to the [[UI-Service]]'s WorldCanvas. The contract: if the screen renders it, the PNG matches.
