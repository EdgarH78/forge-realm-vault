---
Summary: The Review Service frontend — a React SPA whose centerpiece is a hierarchical tree that renders a Map Forge run as Phase → Level → Room → Step, so a reviewer walks the run as the narrative sequence of events the agent actually executed (clarify → plan → readout → asset generation → render loop). A pure presentation primitive (`StageHierarchy` + `AccordionRow`) is fed a `StageNode` tree built from the flat manifest by `buildStageTree`. Each leaf renders a stage body + a `JudgmentRow` that writes tags. Assets appear once per room in a synthesized "Asset generation" node; the render-fix loop renders as a vertical attempt timeline. Backed by [[Review-Service]]; data shapes from [[Review-Capture-Pipeline]].
Tags: #review #tooling #atlasforge #ui
---

# Review Hierarchy UI

The frontend (`apps/review-service/frontend`) is a small React Router SPA. Its design problem: a Map Forge run is a *flat* list of stages in `manifest.json` (`clarify`, `concept-style`, …, `top-down-room-0`, `render`×N, `eval`×N, …), but a reviewer needs to walk it as the *nested narrative* the agent executed. The UI's job is to reconstruct that hierarchy and let the reviewer tag any node.

## Routes (SPA)

Five routes (`App.tsx`):
- `/runs` — `RunsIndex`: all runs across both pipelines.
- `/runs/:runId` — `RunDetail`: bg-removal-retry fixture list.
- `/runs/:runId/:fixtureId` — `AttemptDetail`: the single-asset review (image grid + per-attempt pipeline view + tag controls). This is the Asset Forge surface.
- `/map-forge/:runId/:runDir` — `MapForgeRunDetail`: the hierarchical run review. The centerpiece.
- `/map-forge/:runId/:runDir/clarify` — `ClarifyWizardDetail`: a dedicated drill-in for the clarify Q&A transcript.

## The tree model

`hierarchy/types.ts` defines `StageNode` as a discriminated union:

```
phase           → Planning | Build Map        (top-level container)
  level         → "Level N"                    (Build Map only, multi-level maps)
    room        → "Room: <name>"
      stage     → one MapForgeStageRef leaf (top-down, topology, layout, …)
      assetGeneration → synthesized per-room asset list (see below)
      attemptGroup    → the render→eval→fix-plan→apply-fix loop, grouped
```

`buildStageTree(manifest, …)` turns the flat `manifest.stages[]` into this tree. It's the riskiest piece of the frontend and the most heavily tested — it owns phase classification (`phaseFor`: planning stage-types vs everything-else), level/room grouping, the synthesized asset node, and attempt-group assembly. `StageNode` deliberately uses concrete sentinels over nullable optionals (e.g. `assetGeneration.roomId: ''` for the degenerate single-room fallback, not `string | undefined`) per [[Type-Discipline]].

## Presentation primitive: StageHierarchy + AccordionRow

`StageHierarchy` is presentation-only: it walks a `StageNode` tree and renders nested `AccordionRow`s. It knows nothing about Map Forge semantics — node-kind-specific rendering is injected via callbacks (`renderLeaf`, `renderAttemptGroup`, `renderAssetGeneration`, `renderBetweenTopLevel`), mirroring the renderer-adapter discipline elsewhere in the codebase ([[Architecture-Decoupling]]). `AccordionRow` owns expand/collapse, the status glyph (`pending` / `in-progress` / `completed` / `none`), counter pills, and gray-dimming for pending rows. `collectAllNodeIds` / `findAssetGenerationAncestors` are the tree-walk helpers the route uses to expand a path on demand (e.g. gallery click → expand to the target asset).

The route (`MapForgeRunDetail`) owns all state — selected asset, draft tag text, the asset-tag map, expansion set — and threads it down through the callbacks. The primitive stays stateless.

## Stage bodies + JudgmentRow

Each leaf renders a **stage body** (the captured content) plus a `JudgmentRow` (the tag control: agree/disagree/uncertain + comment, posting to the backend). Stage bodies are content-typed:

- **Planning bodies** (`components/planning/`): `ConceptStyleBody`, `HeroBody` (hero image + palette + materials), `RoomsBody`, `FeaturesBody`, and `ApprovalGateMarker` (the "✓ Plan approved" divider between Planning and Build Map). These parse the LLM `rawResponse` defensively — field-by-field narrowing helpers, Monaco code-view fallback on parse failure — because the captured response is raw model output ([[Type-Discipline]]: narrow at the deserialization boundary).
- **Generic body** (`StageBody`): Monaco JSON view + image strip for stages without a bespoke body. **Stage bodies never render asset cards** — that was the Slice-3 clutter bug; assets live in exactly one place (below).

## Asset generation: one node, one place

A room's generated assets appear in a single synthesized `assetGeneration` node, positioned between the room's `top-down` readout and its render `attemptGroup` — matching the agent's actual event order (readout → generate assets → render). `RoomAssetsList` renders one `EmbeddedAssetCard` per asset (deduped by `fixtureId`, manifest order = generation order). A card is collapsed by default (thumbnail + slug + `JudgmentRow`); expanding lazy-loads `FixtureReviewBody` — the same rich per-attempt pipeline view used by the standalone `AttemptDetail` route. The card picks its data endpoint by presence of `(runId, runDir)`: Map Forge-native per-asset capture, else the bg-removal-retry fallback. The top-level `GlobalAssetGallery` is a flat skim/jump-table; clicking a tile expands the tree path to that asset's in-room card and scrolls it into view. **Single-selection**: at most one card open at a time, driven by the route's `selectedAssetFixtureId`.

This "assets appear once, at the moment generated, never scattered" rule is load-bearing — an earlier design rendered asset cards in every stage body that carried refs and the page became unreviewable.

## The render-fix loop: attempt timeline

The Phase 13 self-eval/fix loop emits per-room, per-attempt stages (`render` / `eval` / `fix-plan` / `apply-fix`, each carrying `attemptIndex`). `buildStageTree` groups these into an `attemptGroup`, rendered by `components/loop/` as a vertical timeline: `PlacementTimeline` (one row per render attempt — badge, image, eval summary), `EvalTimeline` (severity-pilled failure cards), `FixPlanActions` (expandable per-action rows from the apply-fix `actions[]` artifact), `ApplyFixOutcomes` (per-action before/after), and `AttemptConnector` (anchor-jumps between adjacent attempts and their fix plans). `correlation.ts` correlates loop stages to rooms/attempts, with a `resolveAttemptIndex` filename-suffix parser for pre-BE-1 fixtures that didn't stamp `attemptIndex`.

## Live runs

`hooks/useRunPolling` re-fetches a run on a 5s interval while it's in progress (`isRunInProgress(manifest)` — heuristic: `manifest.durationMs === 0` means not yet finished). `pipelineContract.ts` synthesizes `pending:`-prefixed placeholder stage refs for stages the manifest doesn't yet contain, so the tree shows the *expected* full pipeline with not-yet-run steps dimmed; `stageStatus.ts` computes per-stage status and `summarizeProgress` drives a `ProgressBar`. Polling honors `document.visibilityState` and converges automatically once the run completes.

## Boundaries

- Backend APIs, repositories, and tag persistence: [[Review-Service]].
- How the manifest / per-stage JSON / per-asset artifacts are produced: [[Review-Capture-Pipeline]].
- Read the codebase for exact component props and the current `buildStageTree` logic — it changes as the pipeline gains stages.
