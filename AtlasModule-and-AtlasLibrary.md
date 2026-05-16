---
Summary: The plugin architecture. An `AtlasModule` is the unit of feature delivery — a self-contained bundle of modes, panels, services, and selection handlers registered through DI at boot. An `AtlasLibrary` is the same shape minus the UI/mode surface, used for cross-module utilities (CoreUI, Pen tools, VectorLibrary). The `ModuleRegistry` is the central index, and the selection-priority dispatch routes editor sessions to the module that best handles the current selection.
Tags: #ui #plugin #module #di #atlasforge
---

# AtlasModule-and-AtlasLibrary

## Two roles

**`AtlasModule`** (`apps/ui/src/atlasforge/api/ModuleDefinition.ts`) is the feature module. Required: `registerComponents(api, di)` to populate the DI container. Optional (the rest is opt-in):

- `id`, `name`, `description`, `icon`, `type: 'module' | 'library'`, `displayType: ModuleDisplayType.SINGLE_MODE | MULTI_MODE`, `showInLeftNavBar`.
- `getModes(): AtlasMode[]` and `getDefaultModeId(): string` for multi-mode modules.
- `getTopBarUI(): AtlasUIElement` — a persistent top-bar contribution that survives mode changes.
- `onActivate(api, di)` / `onDeactivate(api, di)` — module-level lifecycle hooks.
- Selection trio: `canHandle(SelectionContext)` (cheap capability probe — no side effects, no allocation, no UI), `createEditorSession(ctx, api)`, `getSelectionPriority()`.

**`AtlasLibrary`** (`api/AtlasLibrary.ts`) is the pure DI extension — `{ id, name, register(api, di) }`. No modes, no UI, no selection handling. Used by `CoreUILibrary` (provides ChatUI, TabComponent, ListViewComponent), the Pen library (CirclePen, LinePen, RectanglePen factories), and the `VectorLibrary` (the singleton policy + factory wiring from [[Vector-Math-Extensions]]).

The distinction matters at boot: libraries register first because modules depend on them.

## AtlasMode

`AtlasMode` (`api/AtlasMode.ts`) is what a multi-mode module returns from `getModes()`:

- `id`, `name`, `description`, `icon` — metadata.
- `panelUI: AtlasUIElement` — a floating overlay panel rendered above the canvas while the mode is active.
- `topBarUI: AtlasUIElement` — mode-specific top-bar contribution; replaces the parent module's `getTopBarUI` while this mode is active.
- `onActivate(api, di)` — wires up the canvas tool via `api.setActiveTool(tool)` and/or initializes mode state.
- `onDeactivate(api, di)` — cleans up canvas tools and mode state.
- `subModes?: AtlasMode[]` — reserved for future nested navigation.

The Buildings module, for example, returns modes like `walls`, `floors`, `doors-and-windows`. Each one's `onActivate` swaps in the appropriate canvas tool (`VectorDrawingTool`, `TileBrushTool`, etc.).

## Selection-driven editing

When the user selects items in the canvas, the platform iterates registered modules:

```
for module in modules:
  if module.canHandle?(SelectionContext):
    candidates.push({ module, priority: module.getSelectionPriority?() ?? 0 })
winner = candidates.maxBy(priority)
session = winner.createEditorSession(ctx, api)  // returns { panel, canvasTool?, dispose }
```

This is what lets the Wall module own a wall selection, the Asset Manager own an asset selection, the Map Forge module own its own widget. Priorities follow a convention: higher (10+) for specialized modules, default (0) for general handlers, negative for last-resort fallbacks. Modules don't need to know about each other — the dispatch is by capability + priority.

The returned `EditorSession`:

- `panel: AtlasUIElement` — the inspector panel rendered into the right-side dock through [[FunctionalComponent-and-Renderer]] → [[AtlasUIElementRenderer]].
- `canvasTool?: CanvasTool` — the active canvas tool for the session (e.g. a `VectorEditingTool` for wall vertices).
- `dispose()` — called when the selection changes or clears; mandatory cleanup hook.

## ModuleRegistry

`ModuleRegistry` (`api/ModuleRegistry.ts`) is the central index. `registerModule(m)`, `registerLibrary(lib)`, `getModule(id)`, `getAllModules()`. Exposed through `AtlasForgeAPI.getModuleRegistry()`. Boot wires every module / library into it once; the router and the selection dispatcher both query it.

## The system_modules directory

`apps/ui/src/atlasforge/system_modules/` ships these out of the box:

| Module | Role |
|---|---|
| `core-ui` | Library — provides `ChatUIComponent`, `TabComponent`, `GridViewComponent`, `ListViewComponent`, `BrushSettingsPanel`, plus the canvas-controls subpackage (CanvasButton, CanvasPanel, CanvasEditPanel, VectorEditingTool, VectorEditingSession). |
| `editing` | The selection service + `SelectionCanvasTool`. |
| `pens` | Pen-tool library (line, curved-line, polygon, circle, rectangle, arc). |
| `wall` | Wall editing + door geometry. |
| `floor_tiles`, `terrain`, `shadow`, `stairs`, `objects`, `levels` | One module per [[MapItem-Hierarchy]] kind. |
| `asset_manager`, `asset_forge` | Asset catalog and per-asset workspace. |
| `map_forge` | The AI map-generation UX wired to [[MapForgeAgent]]. |
| `ai_assistant` | Chat UI for the building assistant. |
| `buildings` | Composite editor for connected wall/floor groups. |
| `home`, `loading`, `save`, `export`, `settings`, `map_name`, `map_resize` | Platform UX. |

Each directory is one `AtlasModule` (occasionally one `AtlasLibrary`); each module owns its own modes, tools, and inspector panels.

## The boot flow

```
1. App start → register libraries first (CoreUILibrary, VectorLibrary, PenLibrary, ...)
2. Register modules; each module.registerComponents(api, di) populates DI
3. ModuleRegistry now holds the manifest
4. Router resolves /modules/{moduleId} → activates that module (onActivate)
5. Module exposes its modes; user picks one → mode.onActivate wires canvas tool
6. mode.panelUI renders via [[FunctionalComponent-and-Renderer]] → [[AtlasUIElementRenderer]] → React DOM
7. Selection events → priority dispatch → createEditorSession → inspector panel
```

The hard rule: modules and libraries never import each other directly. Every cross-module dependency goes through DI by string key, and every UI contribution comes back through [[AtlasUIElement]].
