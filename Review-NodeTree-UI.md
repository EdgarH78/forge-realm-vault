---
Summary: The Review Service frontend — a React SPA whose centerpiece is a data-driven LangGraph/LangSmith-style **node tree** that renders a Map Forge run exactly as the `agent_runtime` audit tree shaped it (stage and action nodes, merged & sequence-ordered, no client-side reconstruction). A two-pane layout: a recursive `RunTreeView` on the left, a `NodeDetailPane` (Overview / Inputs / Outputs / Tags) on the right. Artifact bodies load lazily on node selection; text/json render in Monaco, assets in an image viewer + variant filmstrip. Tags anchor to any node via `nodeRef`. The old Phase→Level→Room→Step reconstruction stack (`buildStageTree`, `StageHierarchy`, planning bodies, attempt timeline, asset gallery) was deleted in Phase 6 (2026-06). Backed by [[Review-Service]]; data shapes from [[Review-Capture-Pipeline]].
Tags: #review #tooling #atlasforge #ui
---

# Review NodeTree UI

The frontend (`apps/review-service/frontend`) is a small React Router SPA. Its design problem used to be reconstructing a bespoke hierarchy on the client from a flattened wire contract; the LangGraph redesign (plan `2026-06-06-review-langgraph-redesign.md`, Phases 1–6) deleted that. The UI now renders the `agent_runtime` tree **as-is**: the backend projects stages + actions + artifacts 1:1 ([[Review-Service]] `projectRunTree` / `projectNodeDetail`), and the frontend walks it directly. Adding a new pipeline shape needs no UI change — the tree is generic over any agent run.

## Routes (SPA)

Four routes (`App.tsx`):
- `/runs` — `RunsIndex`: all runs across both pipelines.
- `/runs/:runId` — `RunDetail`: bg-removal-retry fixture list (Asset Forge surface).
- `/runs/:runId/:fixtureId` — `AttemptDetail`: the single-asset review (image grid + per-attempt pipeline view via `FixtureReviewBody`/`AttemptCard` + tag controls). The Asset Forge surface.
- `/map-forge/:runId/:runDir` — `MapForgeRunDetail`: the node-tree run review. The centerpiece.

The bg-removal-retry / Asset Forge surface (`RunsIndex` / `RunDetail` / `AttemptDetail`) is a **separate pipeline** and an explicit fast-follow for tree unification — it was deliberately preserved through the Phase 6 deletion.

## The node model (one rule, no per-pipeline branches)

A tree node is a **stage** OR an **action** (`RunNode = StageNode | ActionNode` in `types.ts`):
- A **stage node**'s children = its child stages **+** its own actions, merged and ordered by `sequence`. Container nodes carry a subtree telemetry rollup (`aggregateTelemetry`).
- An **action node** is a leaf; its content = its `action_artifacts` as `ArtifactDescriptor`s, split by `direction` into Inputs / Outputs.
- **Icon** = lookup on `stageType` / `actionType` (`nodeIcon.tsx`). **Status** = the row's `status` (`statusGlyph.tsx`). **Label** = `name` / `metadata.promptName`.
- **Generic same-type grouping** (presentation only): N sibling actions of the same `actionType` collapse under a count header ("Assets (4)") via `groupSiblingActions.ts`. Uniform across types; not Map-Forge-specific.

## Presentation: RunTreeView + NodeDetailPane

`RunTreeView` is recursive and presentation-only: collapsible rows with indentation guides, selection by node id. The route (`MapForgeRunDetail`) owns all state — selected id, per-node draft tags, the tag list, expansion — and threads it down; selection is tracked **by id** (not by object) because node objects are rebuilt on every poll refetch.

`NodeDetailPane` shows tabs: **Overview** (only when the node has children — telemetry rollup + clickable child timeline via `subtreeStats.ts`), **Inputs**, **Outputs**, **Tags**. Heavy artifact bodies are **fetched lazily** on selection (`getNodeDetail`) — the tree payload carries descriptors only (`hasBody`, asset URLs), keeping the initial load small.

## Artifact rendering

`ArtifactView` dispatches on `kind`: `text`/`json` → Monaco (lazy-loaded; language picked by `monacoLanguage.ts`); `asset` → `ImageViewer` (checkerboard, zoom) + `VariantFilmstrip` (raw / bg-removed / rejected candidates). Asset variants + verdict are folded into the asset `ArtifactDescriptor` server-side — there is no separate gallery (Q2: gallery removed).

For a **container** node the server resolves boundary artifacts (Q1: server-side) — inputs = first subtree action's inputs, outputs = last subtree action's outputs — so the root run node shows prompt-in / final-render-out for free.

## Tagging

`TagBar` (built on the shared `JudgmentRow`: agree / disagree / uncertain + comment) drafts a judgment per node; the top bar's "Save N tags" button POSTs every pending draft via `postMapForgeTag` with the node's `nodeRef` ({kind, id}) + a derived `TagTarget` (`tagTargetFor.ts`), then refreshes from the returned `TagEntry[]`. The `Tags` tab (`TagsTab`) lists a node's existing judgments and deletes by run-level index. Legacy root-anchored tags read back as `nodeRef {kind:'stage', id:rootStageId}` and attach to the root node. `nodeTags.ts` matches tags to nodes and builds the breadcrumb locator.

## Live runs

`hooks/useRunPolling` re-fetches `/tree` on a 5s interval while the run is in progress (`status:'running'`, equivalently `durationMs===0`), honoring `document.visibilityState` and converging once the run completes. The tree reflects partial runs **natively** — open stages carry `status:'running'`; there is **no pipeline prediction** (the deleted `pipelineContract`/`stageStatus` synthesized "pending" placeholders — dropped per Phase 5). A data-driven progress hint counts completed leaf actions over total. A poll refresh merges tree data + tags without disturbing selection or in-flight drafts.

## Boundaries

- Backend APIs, repositories, the tree projection, and tag persistence: [[Review-Service]].
- How the audit tree is *written* by the auditing decorators: [[Review-Capture-Pipeline]].
- Read the codebase for exact component props and the current projectors — the tree is generic, but the icon/label maps grow as the pipeline gains stage/action types.
- UI tech is plain React/Tailwind (Q3: this dev tool stays plain React; AtlasUI declarative elements are for the production UI only).
