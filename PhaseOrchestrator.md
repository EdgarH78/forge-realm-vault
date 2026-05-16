---
Summary: The deterministic sequential phase runner that drives [[MapForgeAgent]]'s execution side. Each phase is a `{name, progressStart, progressEnd, run}` triple; the orchestrator runs them in order, dual-publishes progress on Pub/Sub + a `generation_events` row, bails on the first failure, and re-throws `InsufficientCreditsError` for graceful credit handling. No reflector, no replan — phase order is fixed at the call site. This is the rigid counterpart to [[DeepAgent]].
Tags: #map-forge #phase-state-machine #sse #pubsub #atlasforge
---

# PhaseOrchestrator

## Identity

`PhaseOrchestrator` is constructed once per `handleApprove` call with `(repo, artifactRepo, pubsub, storage, eventsTopic)`. It owns no per-phase state — every value the running phase needs flows in through the `PhaseContext` parameter passed to `run`.

## The contracts

```
PhaseDefinition = {
  name: string,
  progressStart: number,           // 0..1 global progress at phase start
  progressEnd: number,             //              at phase end
  run: (ctx: PhaseContext) => Promise<void>,
}

PhaseContext = {
  generationId, sessionId, userId, ir?, creditService?, scratchpad?,
  repo, artifactRepo, storage,
  emitProgress(progress, message),
  emitRoomProgress(progress, message, roomName, roomIndex, roomCount),
  emitTopDownRoomProgress?(generationId, roomId, roomName, roomIndex, roomCount, stage),
  roomId?, attemptIndex?,          // Phase 13 capture-correlation
}
```

`scratchpad` is the documented untyped escape hatch for cross-phase comms (e.g. `topDownResults` from `TopDownReferencePhase` → re-layout → compilation). It is **not persisted** — anything that must survive a process restart must be an artifact, not a scratchpad entry.

## The main loop

```
runPhases(generationId, sessionId, phases[]):
  for each phase:
    gen = await repo.getGeneration(generationId)
    if !gen || gen.status !== 'active': stop          // user cancelled / earlier failure
    try:
      await repo.updatePhase(generationId, phase.name)
      emit 'phase_started' (DB + pubsub) at progressStart
      build PhaseContext with phase-scoped emitters
      await phase.run(context)
      emit 'phase_completed' (DB + pubsub) at progressEnd
    catch error:
      if error instanceof InsufficientCreditsError: throw  // MapForgeAgent handles
      else: updateStatus='failed', emit 'error', return    // sequential, no rollback
```

Three exit modes: clean traversal of the phase array, `InsufficientCreditsError` re-thrown to [[MapForgeAgent]] for graceful credit handling (which marks the generation `completed` with a partial-results note), or any other throw which marks the generation `failed` and stops.

## The dual event channel

Every progress / lifecycle event is fanned out to **both**:

- `repo.appendEvent(...)` — writes a `generation_events` row in the `agent_runtime` schema (see [[AgentRuntimeSchema]]). Durable, queryable history.
- `pubsub.topic(eventsTopic).publishMessage(...)` — feeds the live SSE stream that the UI listens to.

Both share the same envelope: `{source: 'map-forge', traceId: generationId, generationId, ...data, timestamp}`. The DB write is the *truth*; pubsub is the *liveness signal*. A subscriber that joins mid-generation reconstructs the past from the DB, then attaches to pubsub for new events.

## Progress band convention

Each phase declares `[progressStart, progressEnd]` and emits values inside that band via `emitProgress(p, m)`. The UI maps `(phase, progress)` to a global progress bar without needing to know the phase set in advance.

## Canonical MapForge phase sequence

```
topology              0.10 – 0.25    map-forge-topology → DungeonIR with empty geometry
layout (first pass)   0.25 – 0.45    LLM-driven floorplan; first-pass widths/heights
top-down-reference    0.45 – 0.75    Stage 1 gen → Stage 2 readout (v3 multimodal) →
                                     Stage 2b post-filter → Stage 3 re-layout →
                                     Stage 4 description-driven asset assembly
compilation           0.75 – 0.90    DungeonIRCompiler → SerializedMap; writes
                                     'map-document' artifact
self-eval-and-fix     between        Per-room render→evaluate→fix-plan→apply-fix loop,
   compilation &      3-attempt cap, severity-gated retry policy (D-07)
   validation
validation            0.90 – 1.00    DungeonIRValidator + structural recovery
                                     (LLM-repair → priority-drop → fail)
completed             1.00           Create Map record + Version + 'current' label,
                                     set generation.created_map_id
```

This order is **not** open for the orchestrator to revise — there is no Reflector. Re-ordering or skipping a phase requires editing the call site in `MapForgeAgent.handleApprove`. That's a deliberate trade with [[DeepAgent]]: less flexibility, more legibility, easier to debug in production.

## Failure & recovery semantics

Sequential, no rollback. Artifacts written by phases that succeeded survive the failure of a later phase — the lineage in `parentArtifactIds` is preserved. Restart is the user's job (re-approve regenerates from the latest planning-contract). The phase-by-phase artifact graph is the resume point.
