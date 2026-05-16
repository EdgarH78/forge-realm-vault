---
Summary: Nine topic/subscription pairs wire the service mesh. Every long-running operation has a **request** topic ([[Worker-Service]] subscribes) and an **events** topic ([[API-Service]] subscribes for SSE relay). The "DB write is truth, Pub/Sub is liveness" rule from [[Architecture-Decoupling]] §4 applies — every consumed event also writes a row to `generation_events` so a reconnecting subscriber can reconstruct history. Local dev uses the GCP cloud-sdk emulator on port 8085 with topics auto-created on container start.
Tags: #pubsub #service-mesh #events #atlasforge
---

# PubSub-Topology

## The nine pairs

From `docker-compose.yml`'s `pubsub-setup` container, created exactly once per environment:

| Request topic | Events topic | Purpose |
|---|---|---|
| `atlas-agent-requests` | `atlas-agent-events` | [[MapForgeAgent]] lifecycle: create, message, approve, phase progress, credits_exhausted |
| `atlas-asset-ingest-requests` | `atlas-asset-ingest-events` | Uploaded-asset processing — see [[AssetStorage]] |
| `atlas-asset-build-requests` | *(events flow via image-gen pair)* | Agent-driven asset builds without their own event channel |
| `atlas-image-gen-requests` | `atlas-image-gen-events` | [[AgenticImageGenerationPipeline]] runs |
| `atlas-map-export-requests` | `atlas-map-export-events` | dd2vtt exports |
| `atlas-asset-reprocess-requests` | `atlas-asset-reprocess-events` | sharp re-derivative pipeline (regenerate thumb/preview) |

Each pair has matching subscriptions: `*-sub` appended (e.g. `atlas-agent-requests-sub`, `atlas-agent-events-sub`).

## Envelope shape

Every message — request and event — carries the same outer envelope:

```json
{
  "source": "map-forge" | "asset-ingest" | "image-gen" | ...,
  "traceId":      "<uuid>",
  "generationId": "<uuid>",
  "timestamp":    "ISO 8601",
  ...payload
}
```

`traceId` and `generationId` are usually the same UUID; they're the join key the [[UI-Service]] and the [[API-Service]] SSE relay filter on. The user's JWT is also threaded into request envelopes so [[Worker-Service]] can verify the caller identity before running the handler.

## The request lifecycle

The canonical flow for any long-running operation:

1. **HTTP in.** Client `POST /api/{operation}` to [[API-Service]]. Auth middleware validates the JWT and decorates `request.user`.
2. **Publish request.** API constructs the message envelope (including the user's JWT for downstream verification), publishes to `atlas-X-requests`, returns 202 / `{ ingestId | generationId, status: 'processing' }` synchronously. HTTP round-trip is done.
3. **Worker consumes.** [[Worker-Service]]'s subscription on `atlas-X-requests-sub` receives the message, verifies the JWT (`verifyToken`), dispatches to the handler.
4. **Worker emits events.** The handler publishes progress events to `atlas-X-events` as work proceeds. Every event also writes a row to `generation_events` (or sibling table) in `atlasforge_agents`.
5. **API relays.** [[API-Service]]'s subscription on `atlas-X-events-sub` receives each event, finds matching SSE connections keyed by `generationId`, writes one SSE frame per match.
6. **Client observes.** [[UI-Service]] gets `progress / phase_started / phase_completed / complete / error` and updates its UI.

## Ack semantics

Workers ack messages on completion — success **or** error. The reasoning: errors are observable to the client via published `error` events; redelivering the same message would re-fail the same way. Pub/Sub redelivery is reserved for outright handler crash (the message never gets acked). Handlers must be idempotent under crash redelivery — `AssetIngestionService` re-dedupes by sha256, [[MapForgeAgent]] resumes from the latest artifact, `MapExportProcessor` writes the same output path.

## Durability — DB write is truth

Pub/Sub is the **liveness signal**, not the **history**. Every consumed event is *also* written to a Postgres table (`atlasforge_agents.generation_events` for map-forge; `atlasforge.asset_files` / `assets` indirectly for ingestion). A reconnecting SSE client re-queries the DB for past events, then attaches the live stream. Subscribers that join after the message TTL expired (typically 7 days) will only see new events — older history requires reading the durable table directly.

This duality is why the [[PhaseOrchestrator]] dual-publishes every event (DB write + Pub/Sub publish) and why none of the consumers depend on Pub/Sub replay for correctness.

## Local dev — the emulator

The `pubsub-setup` container in `docker-compose.yml` runs a one-shot Python script against the cloud-sdk emulator on port 8085. It creates all 9 topic + subscription pairs idempotently (catching "already exists"), then `list_topics` / `list_subscriptions` to verify before exiting. Healthchecks gate downstream services (api, worker, image-service) on `pubsub-setup`'s `service_completed_successfully`.

Every Pub/Sub client sees `PUBSUB_EMULATOR_HOST=pubsub:8085` and `GOOGLE_CLOUD_PROJECT=scryforge` in env — the GCP client library auto-routes to the emulator when those are set.

## Prod

Same APIs, real GCP Pub/Sub project. Differences:

- `PUBSUB_EMULATOR_HOST` unset; client uses Application Default Credentials.
- Topics + subscriptions created by Terraform / `cloudbuild.yaml`, not by a setup container.
- Subscription configs: `ackDeadlineSeconds = 600` for the agent and image-gen subscriptions (LLM calls can take minutes); `expirationPolicy.ttl` set to 31 days so subscriptions don't get garbage-collected.
- Dead-letter topic configured on every subscription with `maxDeliveryAttempts = 5` — chronic redeliveries land in a DLQ topic for inspection rather than poisoning the live stream.

## What is NOT in Pub/Sub

- **Streaming LLM output.** Token-by-token gen is handled inline by the worker; only progress events (start, phase, complete) hit Pub/Sub.
- **Asset bytes.** Bytes go directly client → signed-URL → GCS/MinIO. Pub/Sub messages only reference paths.
- **User authentication.** JWTs travel inside message envelopes but the auth system itself is HTTP-only (`forgerealm-auth`).
- **Render-service RPCs.** [[RenderService]] is a synchronous HTTP service; the worker calls it directly. Pub/Sub would add an extra hop for an already-fast operation.
