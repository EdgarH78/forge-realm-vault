---
Summary: Production deploys are triggered by pushes to `main` via GitHub Actions, which submits `cloudbuild.yaml` to Google Cloud Build. Five Docker images build in parallel, then a strict ordering kicks in — Pub/Sub provisioning, DB migrations, agent DB creation, then per-service Cloud Run deploys. Two URL-injection passes wire services together (worker ↔ image-service, api ↔ render-service) because no service knows its own Cloud Run URL until after it's deployed. Auth is GCP Workload Identity Federation (no static keys). Drift between the deploy script and the application code is the dominant failure mode — see [[PubSub-Topology]] for the canonical example.
Tags: #deployment #ci-cd #cloud-build #cloud-run #atlasforge
---

# Deployment

## Trigger

Pushes to `main` only. `.github/workflows/ci.yml`'s `deploy` job runs after `build` (vitest + typecheck + build) and `security` (npm audit + audit-ci) pass. The deploy job authenticates to GCP via Workload Identity Federation (org-level provider `projects/1060995100938/locations/global/workloadIdentityPools/github-actions`), then runs `gcloud builds submit --config=cloudbuild.yaml --substitutions=SHORT_SHA=${GITHUB_SHA::7}`.

PRs do not deploy. Force-pushing or pushing directly to main bypasses no gates beyond the CI workflow itself — there is no manual approval step.

## Pipeline shape (cloudbuild.yaml)

20 steps with explicit `waitFor` dependencies. Parallelism is real where dependencies allow it.

```
build-api ─────────┐
build-worker ──────┤
build-ui ──────────┤     (5× parallel docker build)
build-image-service┤
build-render-svc ──┘
       │
       ▼
push-* (5× parallel; each gated on its build)
       │
       ├──► ensure-pubsub (independent, runs early)
       │
push-api ──► migrate ──► create-agent-db ──► migrate-agents
       │                                            │
       ▼                                            │
deploy-image-service                                │
       │                                            │
deploy-api ◄────────── migrate, migrate-agents, ensure-pubsub
       │
deploy-worker ◄─────── push-worker, deploy-image-service, migrations, pubsub
       │
set-worker-image-service-url (URL injection pass #1)
       │
deploy-render-service ◄── push-render-service, deploy-api
       │
set-api-render-service-url (URL injection pass #2)

deploy-ui (parallel; only needs push-ui)
```

Total wall-clock: ~12–15 minutes. The `deploy-worker` step is the long pole — the worker has the largest image (~1 GiB cold) and the strictest health check (TCP probe on port 8080 within Cloud Run's default 240s startup deadline).

## The three pre-deploy steps

1. **`ensure-pubsub`** — idempotent shell loop over a hardcoded topic/subscription list. Creates missing topics + subscriptions, configures DLQ for request subs. See [[PubSub-Topology]] for the recipe to add a topic. **This is the most common drift source** — application code can add a new subscription without touching this list, and the worker will crash at startup in prod.
2. **`migrate`** — runs `apps/api/src/db/migrate.js` inside the freshly-built API image against the `atlasforge` database. SQL files in `apps/api/migrations/001_*.sql` … forward only.
3. **`create-agent-db` → `migrate-agents`** — derives `AGENT_DATABASE_URL` from `DATABASE_URL` by string-replacing the db name, creates `atlasforge_agents` if missing, then runs the agent migration set from `apps/api/migrations/agents/`. The same migrate.js binary is reused via `MIGRATIONS_DIR` env override.

If any of these fail, the deploy aborts and no Cloud Run revision is created.

## URL injection — why deploys aren't strictly parallel

Cloud Run assigns each service a URL like `https://atlasforge-worker-{hash}-uc.a.run.app`, knowable only **after** the service is deployed. Three services need to know the URL of another at runtime:

- **Worker → Image Service.** `set-worker-image-service-url` runs `gcloud run services update worker --update-env-vars=IMAGE_SERVICE_URL=...` after both are deployed. The worker reads `IMAGE_SERVICE_URL` to call the `[[ImageService]]` sync endpoints.
- **Render Service → API.** `deploy-render-service` looks up the API URL with `gcloud run services describe api` and bakes it into `RENDER_API_BASE_URL` at deploy time. Without this, the headless canvas's `fetch('/api/assets/:id')` falls through to the render-service's own origin (which has no `/api` routes) and silently 404s every asset, producing transparent renders.
- **API → Render Service.** `set-api-render-service-url` does the same in reverse, updating the API after the render-service exists, so `/api/maps/:id/export` can proxy to the render service.

The `update-env-vars` calls implicitly create a new Cloud Run revision per service. So the worker, api, and render-service each get deployed once then immediately rev'd a second time. Both revisions count toward the 1000-revisions-per-service quota; old revisions get garbage-collected after about a week.

## Auth and secrets

- **GCP auth:** Workload Identity Federation maps the GitHub Actions OIDC token to the `scryforge-ci-cd@scryforge.iam.gserviceaccount.com` service account. No static keys are stored in GitHub.
- **Runtime secrets:** Cloud Run pulls from GCP Secret Manager via `--set-secrets`. Current secrets: `ATLASFORGE_DATABASE_URL`, `JWT_SECRET_CURRENT`, `GOOGLE_AI_KEY`, `OPENAI_API_KEY`, `REPLICATE_API_TOKEN`. Migrations also use `ATLASFORGE_DATABASE_URL` (the `availableSecrets` block in cloudbuild.yaml).
- **No `.env` files in prod.** All config is via `--set-env-vars` (non-secret) or `--set-secrets` (secret).

## Health checks and rollback

Cloud Run runs a TCP probe against the service's port on every new revision. If the container doesn't bind within ~240 seconds, the revision is marked failed, traffic stays on the previous good revision, and the cloudbuild step exits non-zero (failing the whole deploy). The user-facing site keeps running on the old revision throughout.

This is the failure mode you see when the worker's startup function throws — e.g. `waitForSubscription` on a missing Pub/Sub subscription, or a `readFileSync` ENOENT on a missing prompt file. The worker process exits with code 1, port 8080 never opens, TCP probe times out, deploy fails. **Always check Cloud Run logs (`gcloud logging read 'resource.type="cloud_run_revision" AND resource.labels.service_name="atlasforge-worker"'`) when this happens — the cloudbuild log only shows "container failed to start", which isn't actionable on its own.**

There is no automatic rollback step beyond Cloud Run's traffic-pin behaviour. To roll back a service after a successful-but-broken deploy, use `gcloud run services update-traffic <service> --to-revisions=<prior>=100`.

## Drift modes — the dominant failure class

Three lists need to stay in sync whenever app code adds a Pub/Sub subscription:

1. The consumer code (`apps/worker/src/index.ts` or similar), with a default subscription name.
2. The local-dev provisioner (`docker-compose.yml` `pubsub-setup` container's `topics_subs` list).
3. The prod provisioner (`cloudbuild.yaml` `ensure-pubsub` step's `topics` and `request_subs` / `event_subs` shell strings).

CI doesn't enforce that these three agree. Local dev tests pass because (1) and (2) match; production deploys silently ship a worker that crashes at startup because (3) lagged behind. See [[PubSub-Topology]] § "Adding a new topic" for the explicit checklist.

Other latent drift surfaces with similar risk: prompt-store YAML files (must be in the image at a path the worker resolves at module-init), JSON schema files (same), and any `readFileSync` at module load. The Dockerized layout puts compiled code at `apps/worker/dist/apps/worker/src/...` — three directories deeper than source — so relative `__dirname` paths that worked in dev silently break in prod. Use `createRequire(import.meta.url).resolve('@atlasforge/<pkg>/package.json')` to anchor file paths to the workspace package instead of the file's location on disk.

## What is NOT in this pipeline

- **Compiled UI bundle to CDN.** [[UI-Service]] is served via the `atlasforge-ui` Cloud Run service (a node serve container), not a static-asset CDN. No CDN-purge step.
- **Smoke tests post-deploy.** The cloudbuild ends after the last service is updated. No probe asserts that `/api/health` returns 200 or that a map renders end-to-end. CI run is green as soon as Cloud Run accepts the revision.
- **Manual approvals.** Push to main = deploy. There is no staging environment between dev and prod.

## Cross-references

- [[PubSub-Topology]] — topic/subscription list, "Adding a new topic" recipe.
- [[Worker-Service]] — the startup function whose throws cause most deploy failures.
- [[Architecture-Decoupling]] § 4 — "DB write is truth, Pub/Sub is liveness" — explains why missing subscriptions can crash startup rather than degrade gracefully.
