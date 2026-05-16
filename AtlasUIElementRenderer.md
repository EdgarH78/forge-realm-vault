---
Summary: The React boundary. A single 1,375-line dispatcher (`apps/ui/src/components/AtlasUIElementRenderer.tsx`) that consumes a pure-primitive [[AtlasUIElement]] tree and emits React JSX. Theme tokens come from an `IUIStyle` singleton resolved via DI from versioned JSON. This file is the only place React knows about AtlasUI primitives — exactly mirroring [[AtlasForgePixiCanvas]]'s role on the canvas side. React version bumps must not cascade past it.
Tags: #ui #react #renderer #adapter #atlasforge
---

# AtlasUIElementRenderer

## Role: the one and only React boundary

`AtlasUIElementRenderer.tsx` is *the* place AtlasUI crosses into React. Sibling files `DialogRenderer.tsx`, `ModePanelRenderer.tsx`, `TopBarRenderer.tsx` are thin parents that mount the tree-rendering function at the right point in the platform layout (modal stack, floating mode-panel slot, top-bar slot); the actual primitive→JSX translation only lives here. Domain code (modules, modes, panels, dialogs from [[AtlasModule-and-AtlasLibrary]]) describes UI as plain [[AtlasUIElement]] data — no `import React`, no JSX, no refs. The renderer translates.

This mirrors the canvas side exactly: see [[AtlasForgePixiCanvas]] for the equivalent PixiJS boundary. Same data-in / pixels-out pattern, different backend.

## The dispatch

The render entrypoint takes a single primitive-only `AtlasUIElement` (produced by [[FunctionalComponent-and-Renderer]]'s multi-pass renderer) and walks it via a large `switch (element.type)`:

```
'label' | 'image' | 'slider' | 'range-slider' | 'button' |
'select-button-group' | 'panel' | 'rich-text' | 'select' |
'text-input' | 'file-input' | 'message-bubble' | 'typing-indicator' |
'status-indicator' | 'scroll-container' | 'chat-ui' | 'accordion' |
'progress-bar' | 'stage-list' | 'horizontal' | 'vertical' |
'border' | 'horizontal-tri' | 'vertical-tri'
```

Anything outside the catalog renders as a debug placeholder — the renderer **never throws on an unknown type**. New primitives are added by extending the [[AtlasUIElement]] union and adding a `case` here; nothing else in the codebase needs to change.

Layout types are handled by a parallel `switch (layout.type)` that converts the abstract slot model into CSS flex / CSS grid. `'vertical'` with `columns: N` produces `display: grid; grid-template-columns: repeat(N, 1fr)`; `'horizontal'` is `flex-direction: row`; `'border'` is a five-region CSS grid; `'horizontal-tri'` and `'vertical-tri'` collapse to a three-slot flex.

## Style resolution — IUIStyle

Theme tokens come from a single DI singleton, key `ui-style`, type `IUIStyle` (`apps/ui/src/atlasforge/ui/style/IUIStyle.ts`). Foundation token groups: `surface { base, raised, overlay, selected }`, `border { default, subtle, focus, selected }`, `text { primary, secondary, muted, link, linkHover, inverse }`, `accent { primary, primaryHover, primaryLight }`, `status { error, warning, success, info }`, `fontSize`, `fontWeight`, `canvas`. Component-level token groups reference foundation through `{token.path}` syntax.

`UIStyleResolver.resolveTheme(rawJson)` makes two passes: pass 1 lifts the foundation groups verbatim; pass 2 walks every other key and substitutes `{path}` references against the foundation map. Themes ship as JSON (`style/dark.json` today). Swap the JSON file at bootstrap and the entire app re-themes — no component code changes.

The renderer resolves `ui-style` once at module-load and threads token values into the converted styles for every primitive that needs them.

## React-specific behavior that does NOT leak into AtlasUIElement

The renderer is the only place these concerns live:

- **Credit-gated buttons** — when `AtlasUIButton.creditCost` is set, the renderer subscribes to `useCreditStore`, reads the user's balance, looks up the action's cost from `atlasforge.action_costs`, and auto-disables the button when the balance is insufficient. The `AtlasUIButton` element doesn't know about the credit store — it only declares its `creditCost.actionType`.
- **CSS conversion** — `getPaddingStyles`, `getBorderStyles`, animation translation from the `animation` record to `animation: name duration timing iteration delay`. None of this leaks back into the data type.
- **DOM event wiring** — `onPointerEnter/Leave/Down/Up/Click` map to React's pointer / mouse handlers; `AtlasTextInput`'s `onChange / onKeyPress / onSubmit` map to React's input event surface.
- **Image placeholders** — `AtlasImage.placeholder` renders a styled div when `src` is absent or fails to load. The element only carries the placeholder text + colors.

## One-way projection

Inputs into the renderer:

- A pure-primitive `AtlasUIElement` tree from [[FunctionalComponent-and-Renderer]]'s multi-pass output.
- The resolved `IUIStyle` from DI.
- Callbacks attached to the element by upstream functional components.

Outputs:

- React JSX rendered into a single mount point (typically the parent `DialogRenderer`, `ModePanelRenderer`, or `TopBarRenderer`).

Nothing flows back from React to the AtlasUI domain. If a component needs to hold state, the state lives in [[FunctionalComponent-and-Renderer]]'s `KeyedHooksRuntime` keyed by `LocalKey` — *not* in React's hook table. React is the projection target; AtlasUI is the source of truth.

## Why this boundary matters

Three concrete payoffs:

1. **React version upgrades don't cascade.** When React 19 changes a hook contract, only this file gets touched.
2. **Renderer alternates are possible.** A hypothetical React Native renderer or a server-side HTML emitter would be a second file matching this dispatch surface — domain code wouldn't know the difference.
3. **Tests can render without React.** The multi-pass renderer's output is plain JSON-ish data; assertions ("the panel contains a button with label 'Save'") can run without spinning up a DOM. Snapshot tests of [[AtlasModule-and-AtlasLibrary]] modules' panels do exactly this.

The boundary rule is the same one that protects the canvas: domain code describes shape + effect, the adapter renders generic primitives, and backend version bumps stop at this file.
