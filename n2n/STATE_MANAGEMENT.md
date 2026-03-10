# STATE_MANAGEMENT.md — N2N Toolz State Architecture

---

## 1. State Library

**Zustand** + **Immer** for all app state.

- Zustand: lightweight, React-hook-based store
- Immer: enables direct mutation syntax in reducers

```typescript
import { create } from 'zustand';
import { immer } from 'zustand/middleware/immer';
import { persist } from 'zustand/middleware';
```

---

## 2. Store Inventory

| Store File | Responsibility | Persisted? |
|---|---|---|
| `graphStore.ts` | Nodes, edges, workflow graph | Yes (auto-save to .n2nflow) |
| `uiStore.ts` | Panels, zoom, theme, selection | Yes (localStorage) |
| `executionStore.ts` | Run state, node statuses, outputs | No |
| `projectStore.ts` | Open project meta, recent files | Yes (localStorage) |
| `pluginStore.ts` | Loaded plugins, node registry | No (rebuilt on load) |
| `historyStore.ts` | Undo/redo snapshots | No |
| `debugStore.ts` | Breakpoints, watch expressions | Yes (per project) |
| `settingsStore.ts` | App preferences | Yes (localStorage) |

---

## 3. graphStore

```typescript
interface GraphState {
  workflowId: string;
  workflowName: string;
  nodes: Map<string, NodeModel>;
  edges: Map<string, EdgeModel>;
  variables: WorkflowVariable[];
  viewport: Viewport;
  isDirty: boolean;             // unsaved changes
}

interface GraphActions {
  addNode:            (node: NodeModel) => void;
  removeNode:         (id: string) => void;
  updateNode:         (id: string, patch: Partial<NodeModel>) => void;
  updateNodeConfig:   (id: string, config: Record<string, unknown>) => void;
  moveNode:           (id: string, position: XYPosition) => void;
  duplicateNode:      (id: string) => string;           // returns new id
  addEdge:            (edge: EdgeModel) => ValidationError | null;
  removeEdge:         (id: string) => void;
  setViewport:        (viewport: Viewport) => void;
  load:               (json: WorkflowJSON) => void;
  serialize:          () => WorkflowJSON;
  markClean:          () => void;
  reset:              () => void;
}
```

**Key behaviors:**
- `removeNode` cascades: removes all edges connected to that node
- `addEdge` validates port compatibility before adding
- `isDirty` set true on any mutation, cleared on save
- Auto-snapshot to `historyStore` on every mutation (debounced 100ms)

---

## 4. uiStore

```typescript
interface UIState {
  // Panels
  nodePaletteOpen:   boolean;
  propertiesPanelOpen: boolean;
  bottomPanelOpen:   boolean;
  bottomPanelTab:    'console' | 'execution' | 'variables' | 'errors' | 'network';

  // Selection
  selectedNodeIds:   string[];
  selectedEdgeIds:   string[];
  hoveredNodeId:     string | null;

  // Canvas
  zoom:              number;
  gridEnabled:       boolean;
  snapEnabled:       boolean;
  minimapVisible:    boolean;

  // Theme
  theme:             'dark' | 'light' | 'system';

  // Modals
  activeModal:       ModalType | null;
  modalProps:        Record<string, unknown>;
}

type ModalType =
  | 'new-project'
  | 'project-settings'
  | 'node-code-preview'
  | 'export'
  | 'deploy'
  | 'shortcuts'
  | 'preferences'
  | 'plugin-install';
```

---

## 5. executionStore

```typescript
interface ExecutionState {
  activeRunId:    string | null;
  runStatus:      RunStatus;
  nodeStatuses:   Map<string, NodeRunStatus>;
  runLog:         LogEntry[];
  runHistory:     RunRecord[];
}

interface LogEntry {
  id:        string;
  runId:     string;
  nodeId?:   string;
  level:     'debug' | 'info' | 'warn' | 'error' | 'trace';
  message:   string;
  data?:     unknown;
  timestamp: number;
}

interface RunRecord {
  runId:      string;
  workflowId: string;
  startedAt:  number;
  endedAt?:   number;
  status:     RunStatus;
  input?:     unknown;
}
```

**Key behaviors:**
- `nodeStatuses` drives canvas node color indicators (green/red/yellow/gray)
- `runLog` feeds Console tab (max 1000 entries, FIFO)
- On `run:complete`, last run saved to `runHistory` (max 50 entries)

---

## 6. historyStore (Undo/Redo)

```typescript
interface HistoryState {
  past:    WorkflowJSON[];   // [oldest → newest], max 100
  future:  WorkflowJSON[];
}

interface HistoryActions {
  push:  (snapshot: WorkflowJSON) => void;
  undo:  () => void;
  redo:  () => void;
  clear: () => void;
  canUndo: boolean;          // computed: past.length > 0
  canRedo: boolean;          // computed: future.length > 0
}
```

```typescript
// Auto-snapshot watcher in graphStore:
graphStore.subscribe(
  state => state.nodes,
  debounce((nodes) => {
    historyStore.getState().push(graphStore.getState().serialize());
  }, 100)
);
```

---

## 7. debugStore

```typescript
interface DebugState {
  breakpoints:       Map<string, Breakpoint>;
  watchExpressions:  WatchExpression[];
  pausedAt?:         { nodeId: string; input: NodeInput };
}

interface Breakpoint {
  nodeId:    string;
  enabled:   boolean;
  condition?: string;     // JS expression, e.g. "input.status === 'error'"
}

interface WatchExpression {
  id:         string;
  expression: string;     // e.g. "outputs.node_001.response.status"
  value?:     unknown;    // populated during run
  error?:     string;
}
```

---

## 8. settingsStore

```typescript
interface SettingsState {
  general: {
    defaultProjectDir: string;
    autoSaveInterval:  number;          // ms (0 = disabled)
    language:          string;
  };
  editor: {
    gridSize:          number;
    snapSensitivity:   number;
    defaultZoom:       number;
  };
  execution: {
    defaultTimeout:    number;          // ms per node
    maxRetryCount:     number;
    sandboxMode:       boolean;
  };
  codeGen: {
    outputDir:         string;
    moduleSystem:      'esm' | 'cjs';
    typescript:        boolean;
    prettier:          boolean;
  };
  appearance: {
    theme:             'dark' | 'light' | 'system';
    fontSize:          number;
    animationSpeed:    'none' | 'fast' | 'normal';
  };
  shortcuts: Record<string, string>;    // action → key combo
}
```

---

## 9. Cross-Store Communication

Stores are independent but communicate via subscriptions:

```typescript
// executionStore listens to WorkflowRunner events
workflowRunner.on('node:success', ({ nodeId, output }) => {
  executionStore.getState().setNodeStatus(nodeId, 'success');
  executionStore.getState().setNodeOutput(nodeId, output);
});

// graphStore dirty → triggers auto-save timer
graphStore.subscribe(
  state => state.isDirty,
  (isDirty) => {
    if (isDirty && settings.autoSaveInterval > 0) {
      scheduleAutoSave();
    }
  }
);
```

---

## 10. React Integration

```typescript
// Component usage pattern
import { useGraphStore } from '../stores/graphStore';

function MyComponent() {
  // Subscribe to specific slice — avoids unnecessary re-renders
  const nodes      = useGraphStore(s => s.nodes);
  const addNode    = useGraphStore(s => s.addNode);
  const isDirty    = useGraphStore(s => s.isDirty);
  // ...
}
```
