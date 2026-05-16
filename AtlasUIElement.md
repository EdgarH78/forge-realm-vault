---
Summary: The framework-neutral discriminated union that describes every UI primitive AtlasForge can render. Plain data — no React, no PixiJS, no behavior beyond event callbacks — so the same tree can render through [[AtlasUIElementRenderer]] to React DOM today and any other backend tomorrow. Layouts compose primitives into trees; the multi-pass renderer in [[FunctionalComponent-and-Renderer]] flattens functional components into pure-primitive trees before render.
Tags: #ui #atlas-ui #primitives #data-shape #atlasforge
---

# AtlasUIElement

## The base shape

Every element is `{ id, type, visible?, disabled?, animation?, onPointerEnter?, onPointerLeave?, onPointerDown?, onPointerUp?, onClick? }`. `type` is the discriminator (`'label' | 'button' | 'panel' | 'slider' | ...`). `id` is mandatory — the multi-pass renderer's `assertUniqueSiblingIds` throws at render time if two siblings clash, surfacing the bug at first paint rather than under user interaction. `animation` carries CSS keyframe metadata (`name`, `duration`, `timing`, `iteration`, `delay`) that [[AtlasUIElementRenderer]] converts into CSS animations.

## The primitive catalog

| Type discriminator | Interface | Purpose |
|---|---|---|
| `'label'` | `AtlasLabel` | Text. `fontSize`, `color`, `fontWeight`, `textTransform`, `letterSpacing`, `lineHeight`, `wordWrap`, `maxWidth`. |
| `'image'` | `AtlasImage` | `src`, `alt`, `width`, `height`, `aspectRatio`, `objectFit`, `title`, optional `placeholder { text, backgroundColor?, textColor? }`. |
| `'button'` | `AtlasUIButton` / `AtlasUIClickButton` | `label`, `glyph`, `tooltip`, `buttonType: 'standard' | 'toggle' | 'checkbox'`, `toggled`, `variant: 'default' | 'primary'`, `badgeText`, and `creditCost { actionType, costLabel? }` for auto-disable on insufficient balance. |
| `'slider'` / `'range-slider'` | `AtlasSlider` / `AtlasRangeSlider` | Bounded value selection; `min`, `max`, `value(s)`, `step`, callbacks. |
| `'text-input'` | `AtlasTextInput` | `placeholder`, `value`, `multiline`, `maxLength`, `inputType: 'text' | 'number' | 'color'`, `step/min/max` for numbers, `onSubmit / onChange / onKeyPress`. |
| `'file-input'` | `AtlasFileInput` | `accept`, `multiple`, `onChange`. |
| `'select'` | `AtlasUISelect` | Dropdown of `AtlasUISelectOption`s. |
| `'select-button-group'` | `AtlasUISelectButtonGroup` | Segmented buttons. |
| `'panel'` | `AtlasUIPanel` | A `layout` (see below) + optional `border`, `padding`, `width`, `height`, `maxHeight`, `backgroundColor`, `fillWidth`, `fillHeight`, and a `hoverOverlay { children, position }` for slide-up reveals. |
| `'accordion'` | `AtlasAccordion` | Collapsible. Items carry `expanded?`, optional `statusElement` (e.g. a status dot), optional `onHeaderClick` side effect. |
| `'scroll-container'` | `AtlasScrollContainer` | A scrollable wrapper. |
| `'rich-text'` | `AtlasRichText` | An ordered `RichTextContent[]` where each segment is `'text' | 'bold' | 'italic' | 'code' | 'link' | 'emoji'`. |
| `'message-bubble'`, `'typing-indicator'`, `'status-indicator'`, `'chat-ui'` | Chat primitives | Wire the AI Assistant module's message timeline. |
| `'progress-bar'`, `'stage-list'` | Progress UI | Long-running operations (export tiling, asset generation). |
| `'dialog'` | `AtlasDialog` | Modal — managed by `DialogManager`. |

`AtlasUIPadding` and `AtlasUIBorder` are non-rendering helpers composed onto `AtlasUIPanel`. Padding accepts either `all` or independent `top/right/bottom/left`. Borders carry `width`, `color`, `style: 'solid' | 'dashed' | 'dotted' | 'beveled'`, plus full + per-corner radius.

## Layouts

`AtlasUILayout` is its own discriminated union — the inner shape of an `AtlasUIPanel`:

- `'vertical'` — flex column; `padding`, `gap`, `align`, `justify`, optional `columns: N` (renders as CSS grid instead of flex), `children: AtlasUIElement[]`.
- `'horizontal'` — flex row; same fields plus `wrap`, `scroll`.
- `'border'` — five-slot region layout: `top`, `left`, `right`, `bottom`, `center`.
- `'horizontal-tri'` — three slots: `left`, `center`, `right` (top-bar style).
- `'vertical-tri'` — three slots: `top`, `center`, `bottom`.

Every layout type carries `id`, `visible?`, `disabled?`, `padding?`, and its specific children/slots. Layouts nest freely — a panel can contain a panel can contain an accordion can contain a panel.

## Plain-data discipline

The single hard rule: an `AtlasUIElement` value contains **no closures over rendering state, no React refs, no PIXI display objects**. The only function-typed fields are event callbacks (`onClick`, `onChange`, etc.), which the renderer wires to native DOM events. Functional components themselves are referenced by their `type` string — registered through DI and resolved by the multi-pass renderer in [[FunctionalComponent-and-Renderer]]. This is what makes the tree printable, serializable in tests, and renderable through any backend that knows the catalog.

## Who produces these trees

[[AtlasModule-and-AtlasLibrary]] modules return `panelUI: AtlasUIElement` (their floating canvas overlay) and `topBarUI: AtlasUIElement` (their top-bar contribution). `getEditorSession(...)` returns a `{ panel: AtlasUIElement, canvasTool?, dispose }` triple. Functional components composed in those panels are themselves `AtlasForgeFC<Props>` functions returning `AtlasUIElement`. Map Forge planning panels, the asset manager's grid, the wall inspector, and every dialog in the app are this same data structure all the way down.
