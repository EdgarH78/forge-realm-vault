---
Summary: The component runtime under `apps/ui/src/atlasforge/ui/`. `AtlasForgeFC<Props>` is a pure function returning an [[AtlasUIElement]]; the `MultiPassRenderer` resolves a tree pass-by-pass until only primitives remain; a `KeyedHooksRuntime` + `KeyedReconciler` give components per-instance state via `LocalKey` — mangled, path-keyed identifiers that are stable under sibling reorder and reset on reparent. The output flows into [[AtlasUIElementRenderer]] for React translation.
Tags: #ui #components #state #renderer #atlasforge
---

# FunctionalComponent-and-Renderer

## The component contract

```ts
type AtlasForgeFC<Props> = (
  props: Props,
  api: AtlasForgeAPI,
  container: AtlasDIContainer,
) => AtlasUIElement
```

A functional component is a pure-ish function that receives its props, the [[AtlasModule-and-AtlasLibrary]] root API, and the DI container, and returns an `AtlasUIElement` tree (often itself containing further functional-component references by `type` string). Side effects belong in event callbacks emitted by the returned element or in module-level `onActivate`/`onDeactivate` hooks, *not* in the render body.

## Registration

`di.registerFunctionalComponent(type, factory)` registers a component under a string `type`. The factory is `{ isSingleton: boolean, create(props?, container?): unknown }`; `createSingletonFactory` / `createTransientFactory` are the typical builders. At render time, the multi-pass renderer asks the DI container for any element whose `type` matches a registered functional component, calls the factory with the element's props, and substitutes the returned `AtlasUIElement` into the tree.

## MultiPassRenderer

`MultiPassRenderer.render(rootElement)` (`ui/runtime/multi-pass-renderer.ts`) loops:

```
currentTree = rootElement
for pass = 0..maxPasses (default 10):
  { resolvedTree, hasFunctionalComponents } = resolvePass(currentTree)
  currentTree = resolvedTree
  if !hasFunctionalComponents: break
warn if maxPasses exhausted
return currentTree
```

`resolvePass` recurses through every nesting point ([[AtlasUIElement]] children, layout `children`, panel `layout.children`, accordion `items[].content`, scroll-container `content`, dialog `content`, the border-layout slots). When it hits an element whose `type` is registered in DI as a functional component, it calls the factory and replaces the element with the result. The "pass" loop exists because a component can return *another* component-typed element, and the renderer is happy to resolve up to ten levels of indirection before giving up.

The result is a tree of **pure primitives** — only the discriminators in [[AtlasUIElement]]'s catalog appear in the final output. That tree is what [[AtlasUIElementRenderer]] then translates into React JSX.

## Keyed state — the load-bearing pattern

The naive "useState by call order" trick breaks under conditional rendering. AtlasUI fixes this with `LocalKey`s:

- `LocalKey = mangle(parent, id) = parent + "%23" + encodeURIComponent(id)`. A path-keyed string. Root: `"my-panel"`. Child: `"my-panel%23slider-1"`. Grandchild: `"my-panel%23slider-1%23track"`.
- `buildVNode(parentKey, element)` (`ui/runtime/vnode.ts`) walks the element tree, mangles every node's id against its parent's localKey, and asserts unique sibling ids via `assertUniqueSiblingIds(children, where)` — duplicates throw at the first build, not at user interaction.

## KeyedHooksRuntime

`KeyedHooksRuntime` (`ui/runtime/hooks.ts`) owns a `Map<LocalKey, ComponentStateBucket>`. Each bucket is `{ localKey, hooks: HookSlot[], onStateChange? }`. The render flow per component:

```
beginComponent(localKey)        // sets the cursor
  useState(initial)             // reads next HookSlot in order, or creates one
  useState(initial)             // ...next slot
endComponent()                  // clears the cursor
```

Hook order **within** a component still matters — that's the in-bucket rule — but cross-component identity is by `LocalKey`, not by render order or call position. Same path ⇒ same bucket ⇒ same state. Different path ⇒ different bucket ⇒ different state.

## KeyedReconciler

`KeyedReconciler.reconcile(newTree, renderCallback)` (`ui/runtime/reconciler.ts`):

1. Extract `LocalKey` sets from the old and new trees.
2. For every key in `old \ new` — `disposeComponent(key)` drops the state bucket (component unmounted, reparented, or simply removed).
3. Register `renderCallback` against every key in the new tree so a future `setState` from inside that component can ping the platform to re-render.

The behavioral implications:

- **Reorder-safe**: a sibling moving from index 2 to index 0 keeps its state (the `LocalKey` includes its `id`, not its index).
- **Reset on reparent**: moving a component under a different parent rebuilds its `LocalKey`, the old bucket is disposed, the new one starts fresh. This is the documented behavior, not a bug.
- **Cheap unmount**: dispose is `O(1)` per key.

## What lives where

The component runtime knows only [[AtlasUIElement]] and DI. It does **not** know about React, PIXI, or any rendering backend. The same component graph could power a different renderer tomorrow by swapping [[AtlasUIElementRenderer]] for another adapter — the function signatures, state buckets, and reconciler stay put.
