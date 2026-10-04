---
Summary: **LLM prompts are versioned YAML the code loads by NAME, never by version** (`getLatestPrompt('map-forge-planner-levels')`), so the version line is a changelog nobody reads at runtime. That makes the editing rule simple and easy to get backwards: **add a version when what the model is asked to PRODUCE changes; edit in place when the prompt restates something the CODE owns and that thing changed.** Selection is the numeric max of `vN.yaml` **filenames**, while `metadata.version` is copied through unvalidated — the two can disagree, and three shipped prompts did. Prose that restates a code constant must be pinned by a test or it drifts; the halves that are pinned survive and the halves that are not go quietly false.
Tags: #prompts #agent-pipeline #prompt-store #atlasforge
---

# Prompt Versioning

Prompts live in `packages/prompt-store/prompts/<name>/vN.yaml` and reach code only through `PromptStore.getLatestPrompt(name)`. Code never names a version. So a new version is not a deployment event — it is simply the new latest, and the old one becomes history that nothing loads. That cheapness is what makes the add-or-edit decision worth having a rule for: both options are one small diff, and only one of them is right.

## The rule

**Add a version** when what the model is asked to **produce** changes — a new constraint it must satisfy, a different reply shape, an instruction that changes its judgement. The test: *would a run from last week need to be separately attributable from a run today?* If yes, the old contract has to stay readable at its own number.

**Edit in place** when the prompt restates a fact the **code** owns and that fact has changed. The model's task is unchanged; the prompt was merely wrong about the world. Numbers the code derives, downstream consequences, mechanism descriptions.

Worked examples, all from the room-geometry and level-stack work (2026-09 → 2026-10):

| change | call | why |
|---|---|---|
| `map-forge-planner-levels` v6 — defer the mixed building/dungeon stack | **new version** | the model is asked for a different reply; v5's contract must stay attributable |
| the `xs` tier row in three planner prompts | **in place** | prose restating `ROOM_SHAPE_TABLE`, which the code owns |
| `map-forge-planner-levels` "a count cap keeps the FIRST entries" | **in place** | a statement of what `planLevels` does downstream, not an instruction — correcting it changes nothing about what the model is asked for, so it needs no capture |
| `map-forge-layout` "the level falls back to a dungeon" → "the WHOLE MAP falls back" | **in place** | same: wrong about the consumer |

**The bump is only free because nothing pins a version.** Map Forge's activities all call `getLatestPrompt`, and no test or fixture asserts a version number for them. `labels.yaml` *can* pin one (40 prompt directories carry one), and the moment a label or a pinned consumer points at a version, "add vN" stops being a no-op deploy and becomes a change to whatever reads the label.

## Superseded versions are history — with one exception

Do not edit an older version's prose. Its job is to record what was asked at the time, which is the whole reason the new-version half of the rule exists.

The exception is **metadata that disagrees with the filename**, because that is a bug rather than a record. `image-gen/doors/image/v1.yaml` declared `version: "7"`, `floortiles` declared `"5"` and `terrain` declared `"6"` — artifacts of #369, which renamed those files into the namespaced layout while keeping numbers from the old flat scheme. No `v7.yaml` ever existed at that path. Corrected 2026-10-04.

## Mechanics that bite

**Selection is by filename, metadata is unvalidated.** `getPromptVersions` reads `vN.yaml` names and `getLatestPrompt` takes the numeric max (`parseInt`, not a string sort — so v10 beats v9). But `promptYamlToPrompt` copies `metadata.version` through with no validation, so a `v6.yaml` declaring `"5"` is *served as v6 and reported as v5* — the orchestrator would log and audit a version that is not the one running. Guarded since 2026-10-04 by `every prompt that ships` in `FileBasedPromptStore.test.ts`, which cross-checks every declared version against the versions its directory actually offers.

**A malformed prompt used to fail silently in the one place that noticed.** Each `*PromptLoad.test.ts` parses its YAML in the `describe` body, so a parse error is a file-level *collection* failure: vitest reports it outside the per-test summary and the file's tests vanish from the count. A broken `v6.yaml` reached a green-looking "167 passed" that way. The sweep above now fails by name instead, for every prompt and every tool. `FileBasedPromptStore.listPrompts` re-throws by design; `FileBasedToolStore.listTools` used to swallow with a bare `// Skip invalid tools` and now matches it.

**Write `description` as a block scalar** (`>-`). A double-quoted scalar with embedded quotes is what broke that v6, and the hazard disappears entirely with `>-`. `created` self-dates per version; `updated` tracks in-place edits.

## Prose restating a code constant must be pinned

This is the recurring failure, and the evidence is unusually clean. `map-forge-planner-levels`' ordering paragraph stated two consequences of its top-to-bottom contract: elevation derives from array position, and a count cap keeps the first entries. The elevation half was pinned by a test; the cap half was not. When truncation changed to keep the window containing the entrance, the pinned half stayed true and **the unpinned half went false and silent**.

So: if a prompt restates a number, a table or a behaviour the code owns, pin it. See [[RoomGeometry]] for the fullest worked case — a restatement checklist naming every site and which are machine-checked — and the drift tests in `apps/orchestrator/src/activities/__tests__/` (`tierVocabularyDrift`, `layoutShapeMenuDrift`, `fillContractDrift`) for the pattern.

## Deployment

**Local compose serves prompt YAML live.** `./packages` is bind-mounted into both worker and orchestrator and `PROMPTS_PATH` points at the source tree, not a built `dist`; `getPromptVersions` re-reads the directory per call and the cache keys on the resolved `name@version`. So adding a `vN.yaml` takes effect without a rebuild *or* a restart. **Deployed images bake `packages/`**, so reaching prod needs a rebuild and deploy. See [[Architecture-Decoupling]] §7, which said the opposite until 2026-10-04 and sent people into a needless rebuild on every prompt iteration.

## The cost, accepted

`map-forge-planner-levels` is now six files (58 / 68 / 113 / 153 / 157 / 191 lines) — the early ones grew as the contract did, but v4 → v6 are near-duplicates, so the next contract change re-copies ~190 lines to change ten. That is the price of the rule and it is worth paying: rollback is deleting one file, and a run from any date can be read against the text it actually used. What makes it tolerable is that only the *contract* half of the rule forks a file — fact corrections stay in place, and they are the majority.

Related: [[Architecture-Decoupling]] (prompts as the versioned seam), [[RoomGeometry]] (the restatement checklist), [[MapForgeAgent]] and [[Orchestrator-Service]] (the consumers), [[Type-Discipline]] (the unvalidated `as T` at the YAML boundary that lets `metadata.version` lie).
