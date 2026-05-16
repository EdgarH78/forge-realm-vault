---
Summary: The generic plan-reflect-execute loop in `packages/agents/src/platform/`. State is a user-defined `S`; a Planner produces a Plan, Specialists execute steps with ToolRegistry-mediated state mutations, and a Reflector updates the plan between steps. Snapshot-based — persistence is delegated. NOT currently used by map generation; that path runs through [[PhaseOrchestrator]].
Tags: #agent #platform #plan-reflect #state-machine #atlasforge
---

# DeepAgent

## Identity

`DeepAgent<S>` is the platform-level generic agent. `S` is the consumer's state type — for the `PromptAssistantAgent` runtime, `S` is `PromptAssistantState`; new agents can pick anything. Construction takes a `Planner<S>`, a `Reflector<S>`, a `SpecialistRegistry<S>`, a `ToolRegistry<S>`, an `initialState: S`, and a config `{ maxSteps = 12, autoFinishOnAllDone = true }`. The agent owns a single `DeepAgentSnapshot<S>` internally; `loadSnapshot` / `getSnapshot` are the persistence seams — actual storage is the caller's job (see [[AgentRuntimeSchema]] for the production wiring).

## The Plan shape

```
Plan = { id, goal, steps: PlanStep[], currentStepIndex, notes, done }
PlanStep = { id, description, assignee?, status, resultSummary?, error?, inputs? }
PlanStepStatus = 'pending' | 'in_progress' | 'done' | 'skipped' | 'failed'
```

`assignee` is the name of a `SpecialistFn` registered in the `SpecialistRegistry`. The orchestrator throws on first missing assignee or unregistered specialist with an explicit error listing what *is* registered — surface bugs at boot, not under load.

## The Specialist contract

```
SpecialistFn<S> = ({step, state, tools}) => Promise<{ result, stateUpdate? }>
StateUpdate<S> = { patch?: Partial<S> } | { reduce?: (prev: S) => S }
```

A specialist sees the current step, the current state, and the tool registry. It returns its result (logged into history) plus an optional `StateUpdate`. `patch` is a shallow merge; `reduce` is the escape hatch for full control. The default reducer (when `createAgent`/`createPromptAgent` get none) appends the result into `state[name]` as an array.

## The Tool contract

```
ToolFn<S> = (args: JsonObject, state: S) => S | Promise<S>
```

Tools are purely functional state-in / state-out. They do not call LLMs; specialists do. The split exists so an LLM-bound specialist can decide which tool to call by name, and the tool executes deterministically against the current state — easier to test, easier to reason about state evolution.

## The runDeep loop

```
runDeep({ goal, context }):
  ensure snapshot (create plan if absent or plan.done)
  while !plan.done && stepsExecuted < maxSteps && hasPendingStep(plan):
    step = getNextPendingStep(plan)  // first pending at/after cursor
    step.status = 'in_progress'
    { result, stateUpdate } = await specialists[step.assignee]({step, state, tools})
    step.status = 'done'; step.resultSummary = summarize(result)
    state = applyStateUpdate(state, stateUpdate)
    { plan, stateUpdate } = await reflector.reflect({goal, plan, step, observation: result, state, availableSpecialists})
    state = applyStateUpdate(state, reflectUpdate)
    advanceCursor(plan)
    if autoFinishOnAllDone && !hasPendingStep(plan): plan.done = true
    history.push({stepId, result, stateAfter: state})
    snapshot = { plan, state, history, lastResult: result }
```

Three termination conditions: `plan.done`, `maxSteps` exceeded, or no pending steps remain. The reflector may insert / re-status / re-order steps, but cannot mutate state directly — only through the returned `stateUpdate`.

## Factories: createAgent / createPromptAgent

`createAgent<S, R>({ name, callPrompt, reducer? })` wraps any async function as a specialist. `createPromptAgent<S, R>({ name, prompt, buildVariables, mapResult?, reducer? })` wraps an `LLMPrompt` from `@atlasforge/prompt-store`: it builds the prompt variables from the current step/state/tools, invokes the prompt, distills the `LLMPromptResult` (default: `.json` if JSON-mode, else `.text`), and applies the reducer. Both default to `state[name] = [...state[name] ?? [], result]` when no reducer is supplied.

## Where it lives — and doesn't

`DeepAgent` is the runtime under `PromptAssistantAgent` (the prompt-editing assistant in `packages/agents/src/runtimes/PromptAssistantAgent/`). It is **not** the runtime under [[MapForgeAgent]] — map generation uses [[PhaseOrchestrator]], a more rigid sequential state machine where phase order is fixed at the call site and no reflector revises the plan. The two architectures coexist: DeepAgent is the open-ended deep loop, PhaseOrchestrator is the deterministic production pipeline.
