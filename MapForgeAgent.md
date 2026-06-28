---
Summary: SUPERSEDED. The in-process `apps/worker/src/agents/MapForgeAgent.ts` orchestrator for AI map creation was DELETED in the ADK+Temporal migration — map-forge now runs as the `mapForgeParentWorkflow` nested workflow tree in [[Orchestrator-Service]] (cutover #611, agent code deleted #613). The clarify→plan→approve→execute lifecycle is preserved: `create`/`message`/`approve` arrive as Pub/Sub→Temporal signals; the 4-step planner + 7-phase execution became parent→level→room→asset child workflows + ADK activities. Kept as a redirect because many notes still link [[MapForgeAgent]].
Tags: #map-forge #orchestrator #agent #superseded #atlasforge
---

# MapForgeAgent  *(superseded)*

> **Status: SUPERSEDED — code deleted.** `MapForgeAgent` and the whole `apps/worker/src/agents/` tree were removed in the ADK+Temporal migration (map-forge cutover #611; dead-code deletion #613). This note is a redirect; the live architecture is [[Orchestrator-Service]].

## What replaced it
- **The three entry points** (`handleCreate` / `handleMessage` / `handleApprove`) are now the `mapForgeParentWorkflow` started by the [[PubSub-Topology]] ingress: a `create` message → `client.workflow.start` (`workflowId = generationId`); `message` / `approve` → Temporal **signals**. See [[Temporal]].
- **The 4-step planning pipeline** (clarify → concept → levels → rooms, with the hero-image fan-out) is now a set of ADK-agent **activities** (`assessMapIntent`, `planConcept`, `planLevels`, `planRooms`, `genHeroImage`) called from the workflow.
- **The 7-phase execution** that handed off to [[PhaseOrchestrator]] is now **nested child workflows**: parent → `levelWorkflow` (per level) → `roomWorkflow` (per room) → `assetForgeWorkflow` (per unique asset) → `compileRoom`, with a map-wide dedup barrier before the asset fan-out.
- **Artifact lineage + the `agent_runtime` audit tree** are unchanged as the system-of-record, and the SSE envelope is byte-identical — so the API/UI contract held across the cutover.

The determinism rules that now govern this code are [[Temporal-Workflow-Activity-Boundary]]; the full migration is `.ai/plans/2026-06-09-adk-temporal-migration/`.
