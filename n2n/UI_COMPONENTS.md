# UI_COMPONENTS.md — N2N Toolz UI Component Reference

---

## 1. Component Architecture

```
src/components/
├── canvas/          ← Canvas and node rendering
├── panels/          ← Sidebar panels
├── menus/           ← Menu bar and context menus
├── modals/          ← Dialog windows
├── nodes/           ← Per-node custom renderers
├── debug/           ← Debug-specific components
└── shared/          ← Reusable primitives
```

---

## 2. Canvas Components

### CanvasArea

| Prop | Type | Description |
|---|---|---|
| `workflowId` | `string` | Active workflow ID |
| `onNodeAdd` | `(node: NodeModel) => void` | Drop from palette |
| `readonly` | `boolean` | Disable editing |

**Behavior:** Wraps `ReactFlow`, handles drag/drop from NodePalette, registers node types from NodeRegistry.

---

### NodeRenderer

Renders a visual node on the canvas. Each `NodeDefinition` can register a custom renderer:

```typescript
interface NodeRendererProps {
  id:             string;
  data:           NodeModel;
  selected:       boolean;
  dragging:       boolean;
}
```

**Default NodeRenderer regions:**

```
┌─────────────────────────────┐
│  [StatusDot] [Icon] Label   │  ← NodeHeader
├─────────────────────────────┤
│  ○ input1     output1 ○     │  ← PortRow (inputs left, outputs right)
│  ○ input2     output2 ○     │
├─────────────────────────────┤
│  [inline config fields]     │  ← NodeInlineConfig (optional, collapsible)
└─────────────────────────────┘
```

---

### NodeHeader

| Prop | Type | Description |
|---|---|---|
| `label` | `string` | Node display name |
| `icon` | `string` | SVG src |
| `color` | `string` | Header background hex |
| `status` | `NodeRunStatus` | Colors status dot |
| `disabled` | `boolean` | Shows dimmed style |
| `breakpoint` | `boolean` | Shows breakpoint indicator |
| `onLabelChange` | `(v: string) => void` | Inline rename |

---

### PortHandle

| Prop | Type | Description |
|---|---|---|
| `id` | `string` | Port name |
| `type` | `'source' \| 'target'` | React Flow handle type |
| `dataType` | `DataType` | Colors the port dot |
| `label` | `string` | Label text |
| `required` | `boolean` | Shows asterisk |
| `connected` | `boolean` | Filled dot when connected |

---

### EdgeRenderer

Custom edge with animated pulse during execution:

```typescript
interface EdgeRendererProps {
  id:           string;
  source:       string;
  target:       string;
  sourceHandle: string;
  targetHandle: string;
  data: {
    animated:  boolean;
    label?:    string;
    dataType?: DataType;
    hasError?: boolean;
  };
}
```

---

## 3. Panel Components

### NodePalette

| Prop | Type | Description |
|---|---|---|
| `onNodeDragStart` | `(def: NodeDefinition) => void` | Fires when user starts dragging |

**Internal State:**
- `searchQuery: string`
- `expandedCategories: Set<string>`
- `sortMode: 'alpha' | 'recent'`

**NodePaletteItem:**
```
[Icon] Label
       [Category badge]       [Drag handle]
```

---

### PropertiesPanel

Context-sensitive. Renders different content based on selection:

| Selection State | Content Rendered |
|---|---|
| Nothing | `CanvasProperties` (grid, zoom) |
| One node | `NodeProperties` |
| Multiple nodes | `MultiSelectProperties` (bulk enable/disable) |
| One edge | `EdgeProperties` |

**NodeProperties sub-sections:**
- `NodeInfoSection` — name, type, ID
- `NodeConfigForm` — dynamic form from `configSchema`
- `PortListSection` — read-only port display
- `ExecutionOptionsSection` — timeout, retry, continueOnError
- `NodeNotesSection` — markdown textarea

---

### NodeConfigForm

Dynamically renders form fields from a node's `configSchema`:

| FieldType | Component |
|---|---|
| `string` | `<TextInput>` |
| `number` | `<NumberInput>` |
| `boolean` | `<Toggle>` |
| `select` | `<Select>` |
| `multiselect` | `<MultiSelect>` |
| `textarea` | `<TextArea>` |
| `code` | `<MonacoEditor>` (inline, small) |
| `json` | `<JSONEditor>` |
| `secret` | `<SecretInput>` (masked) |
| `expression` | `<ExpressionInput>` (syntax highlight) |
| `file-path` | `<FilePathPicker>` |
| `keyvalue` | `<KeyValueEditor>` |

---

### BottomPanel

| Tab | Component | Content |
|---|---|---|
| Console | `ConsoleTab` | Scrolling log stream with level filters |
| Execution | `ExecutionLogTab` | Node execution timeline list |
| Variables | `VariablesTab` | Runtime variable tree view |
| Errors | `ErrorsTab` | Error list with stack traces |
| Network | `NetworkTab` | HTTP request log |

---

## 4. Menu Components

### TopMenuBar

Pure presentational — actions dispatched to stores or via IPC.

```typescript
interface TopMenuBarProps {
  projectName:  string;
  isDirty:      boolean;
  runStatus:    RunStatus;
}
```

---

### ContextMenu

Rendered via Radix UI `ContextMenu`. Three variants:
- `CanvasContextMenu` — appears on canvas right-click
- `NodeContextMenu` — appears on node right-click
- `EdgeContextMenu` — appears on edge right-click

---

## 5. Modal Components

| Modal | Trigger | Content |
|---|---|---|
| `NewProjectModal` | File → New Project | Name, directory picker, template |
| `ProjectSettingsModal` | File → Project Settings | Name, description, env vars |
| `ExportModal` | File → Export | Target selector, options |
| `DeployModal` | Deploy → Deploy | Target selector, build log |
| `NodeCodePreviewModal` | Node right-click → View Code | Monaco editor (read-only) |
| `PluginInstallModal` | Plugins → Install | Search registry, install button |
| `ShortcutsModal` | Help → Shortcuts | Keybinding table |
| `PreferencesModal` | Edit → Preferences | Settings form |

---

## 6. Debug Components

### BreakpointIndicator

Overlaid on node header when breakpoint is set:

```
🔴  (red dot, top-left corner of node)
⏸️  (pause icon when paused at this node)
```

### ExecutionTimeline

```
[node label] ────── 12ms ───── ✅
[node label] ────────────── 340ms ──── ❌
[node label] ─ 3ms ── ✅
```

### VariableTree

Recursive tree of runtime variable scope. Editable in dev mode:

```
▼ workflow
    ▼ n_a1b2_out
        status: 200
        data: { ... }
    n_c3d4_out: "parsed string"
```

---

## 7. Shared Primitives

| Component | Props | Description |
|---|---|---|
| `Badge` | `label, variant` | Status/type labels |
| `Tooltip` | `content, children` | Hover tooltips |
| `Icon` | `name, size, color` | SVG icon renderer |
| `Spinner` | `size` | Loading indicator |
| `Toast` | `message, type, duration` | App-level notifications |
| `Divider` | `orientation` | Visual separator |
| `Kbd` | `keys` | Keyboard shortcut display |
| `CodeBlock` | `code, language` | Syntax-highlighted code |
| `MonacoEditor` | `value, language, onChange, readonly` | Full Monaco instance |
| `SecretInput` | `value, onChange` | Masked text + show toggle |

---

## 8. Node Status Color Mapping

| Status | Canvas Color | Description |
|---|---|---|
| `pending` | Default (gray header) | Not yet executed |
| `running` | Blue pulse animation | Currently executing |
| `success` | Green glow | Completed successfully |
| `error` | Red + error badge | Failed |
| `skipped` | Dimmed (50% opacity) | Disabled or bypassed |
| `retry` | Orange + retry count | Retrying after failure |
| `breakpoint` | Red dot indicator | Breakpoint set |
| `paused` | Yellow highlight | Paused at breakpoint |
