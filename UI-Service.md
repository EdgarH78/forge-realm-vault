---
Summary: The browser-side React + Vite + PIXI client. Port 5173 in dev. Mounts the AtlasUI plugin architecture, renders maps through [[AtlasForgePixiCanvas]], and talks to [[API-Service]] over HTTP + SSE. Authentication delegates to a dev-auth shim locally and to `forgerealm-auth` in prod; every request carries `Authorization: Bearer <jwt>`. The UI never touches [[PubSub-Topology]] directly — all eventing is SSE through the API.
Tags: #service #ui #frontend #atlasforge
---

# UI-Service

## Stack & layout

`apps/ui/` is a Vite 5 + React 18 + TypeScript 5.5 SPA. Bundled with Tailwind 3, Zustand 4, PIXI 8, react-router-dom 6, Immutable.js, @flatten-js/core, bezier-js. Build inputs in `vite.config.ts` are `main` (the editor) and `headless` (a stripped-down bundle the [[RenderService]] loads inside Chromium). Production target: Cloud Run via `Dockerfile.ui` serving on port 8080; dev target: `npm run dev` binding `0.0.0.0:5173` with a proxy forwarding `/api`, `/agent`, `/healthz` to `VITE_API_HOST:VITE_API_PORT`.

## Bootstrap

`apps/ui/src/main.tsx` mounts `<AppRoutes />`. During app start:

1. Token capture — `/auth/callback?token=…` route stashes the JWT (in memory + sessionStorage).
2. DI bootstrap — `VectorLibrary`, `MapItemFactory`, and every system module under `apps/ui/src/atlasforge/system_modules/*` register through [[AtlasModule-and-AtlasLibrary]] against the `AtlasDIContainer`. Libraries register first (`CoreUILibrary`, `PenLibrary`, `VectorLibrary`); feature modules register second.
3. Module routes — `/modules/{moduleId}` resolves to `ModuleRoute.tsx`, which activates the module via `module.onActivate(api, di)`, exposes its modes, and renders the active mode's `panelUI` through [[FunctionalComponent-and-Renderer]] → [[AtlasUIElementRenderer]].
4. Canvas — `WorldCanvasMVVM.tsx` owns the PIXI `Application` and instantiates exactly one [[AtlasForgePixiCanvas]]. Domain map state lives in [[Map]] / [[Layer-and-Level]] / [[MapItem-Hierarchy]] and is rendered through the canvas's projection pipeline.

## Auth flow

Local dev: click "Sign In" → 302 → dev-auth container on `:3099` → POST `/login` → HS256-signs a JWT for `user 00000000-…001` with `iss=forgerealm-auth, tier=master, exp=now+14400s` → 302 to `/auth/callback?token=…`. The same JWT format is used in prod; only the issuing service changes (real `forgerealm-auth` instead of the dev shim).

Every fetch through the API client attaches `Authorization: Bearer <token>`. Responses carrying `X-Refreshed-Token` (when the token is within 30 minutes of expiry) replace the in-memory token in place.

## Render targets

Three independent render surfaces, all sharing the same DI container:

- **Canvas** — `WorldCanvasMVVM` → [[AtlasForgePixiCanvas]] → PIXI display tree. The only file in the bundle that imports `pixi.js`.
- **Chrome (panels, dialogs, top bar)** — modules return `panelUI: AtlasUIElement` and `topBarUI: AtlasUIElement` from [[AtlasModule-and-AtlasLibrary]]; the multi-pass renderer flattens them; [[AtlasUIElementRenderer]] translates to React JSX rendered through `DialogRenderer`, `ModePanelRenderer`, `TopBarRenderer`.
- **Headless** — `apps/ui/headless/index.html` is a minimal entry that boots WorldCanvas only (no chrome, no input controllers). [[RenderService]] loads it as `/headless/?headless=1` and drives renders via `window.__renderMap`.

## SSE consumption

The UI subscribes to API-side SSE endpoints scoped by `generationId`:

- `/api/mapForge/{generationId}/events` — [[MapForgeAgent]] lifecycle (planning_contract, planning_progress, phase_started, progress, phase_completed, credits_exhausted)
- `/api/assets/ingest/{ingestId}/events` — asset upload progress
- `/api/imageGeneration/{generationId}/events` — agentic gen progress
- `/api/map-export/{generationId}/events` — dd2vtt export progress

Reconnect-resume reconstructs missed events by re-querying the API; see [[PubSub-Topology]] for the underlying durable-history rule.

## What the UI is NOT

It is **not** a Pub/Sub consumer; **not** a JWT issuer; **not** an asset bytes mover (signed URLs go directly to GCS/MinIO via PUT). It is also not the only render target — the [[RenderService]] runs the same Vite bundle headless inside Chromium for evaluation renders during [[MapForgeAgent]]'s SelfEvalAndFixPhase. Anything that must render identically in both surfaces must be deterministic, which is exactly why [[Architecture-Determinism]] and [[Vector-Math-Extensions]] are written the way they are.
