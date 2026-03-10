# ARCHITECTURE.md — N2N Toolz System Architecture

---

## 1. Architecture Principles

| Principle | Implementation |
|---|---|
| Visual-First | Every Node.js construct has a visual equivalent |
| Code Fidelity | Generated code is human-readable, linted, and production-safe |
| Extensibility | All nodes are defined by schema — core and plugins share identical contract |
| Isolation | Each workflow run is sandboxed; no shared state between runs |
| Determinism | Same graph always produces same code output |

---

## 2. Six-Layer System

```
┌──────────────────────────────────────────────────────────────────┐
│  LAYER 1: UI LAYER                                               │
│  React 18 + TypeScript                                           │
│  Canvas (React Flow) | Node Palette | Properties | Console       │
├──────────────────────────────────────────────────────────────────┤
│  LAYER 2: GRAPH ENGINE LAYER                                     │
│  Node Registry | Edge Manager | Serializer | topoSort            │
├──────────────────────────────────────────────────────────────────┤
│  LAYER 3: CODE GENERATION LAYER                                  │
│  Graph Analyzer | AST Builder | Scope Resolver | Code Emitter    │
├──────────────────────────────────────────────────────────────────┤
│  LAYER 4: RUNTIME / EXECUTOR LAYER                               │
│  WorkflowRunner | NodeExecutor | Sandbox VM | DebugInterceptor   │
├──────────────────────────────────────────────────────────────────┤
│  LAYER 5: PLUGIN LAYER                                           │
│  PluginLoader | Custom NodeRegistry | Schema Validator           │
├──────────────────────────────────────────────────────────────────┤
│  LAYER 6: PERSISTENCE LAYER                                      │
│  fs/promises | ProjectStore | UndoHistory | Preferences          │
└──────────────────────────────────────────────────────────────────┘
```

---

## 3. Process Architecture (Electron)

```
┌─────────────────────────────┐     IPC Bridge     ┌────────────────────────────┐
│     MAIN PROCESS            │◄──────────────────►│    RENDERER PROCESS        │
│                             │                    │                            │
│  - File System access       │                    │  - React UI                │
│  - Plugin loading (ESM)     │                    │  - Canvas Engine           │
│  - Code generation          │                    │  - State (Zustand)         │
│  - Build & deploy           │                    │  - Properties panels       │
│  - Workflow runtime         │                    │                            │
│  - Electron window mgmt     │                    │                            │
└─────────────────────────────┘                    └────────────────────────────┘
           │
           ▼
  ┌──────────────────┐
  │  CHILD PROCESSES │
  │  - Dev server    │
  │  - Build runner  │
  │  - SSH deployer  │
  └──────────────────┘
```

---

## 4. Data Flow: Design → Execution

```
[User Interaction on Canvas]
        ↓
[Graph Store mutation (Zustand)]
        ↓
[GraphSerializer → WorkflowJSON]
        ↓
     ┌──────────────────────────┐
     │                          │
     ▼                          ▼
[WorkflowRunner]          [CodeGenEngine]
(live execution)          (export mode)
     │                          │
     ▼                          ▼
[NodeExecutor per node]   [ASTBuilder → CodeEmitter]
     │                          │
     ▼                          ▼
[Output → executionStore] [dist/index.js + routes/]
```

---

## 5. Graph Engine Internals

### 5.1 Node Registry

```typescript
class NodeRegistry {
  private nodes: Map<string, NodeDefinition> = new Map();

  register(def: NodeDefinition): void;
  get(type: string): NodeDefinition;
  getAll(): NodeDefinition[];
  getByCategory(cat: string): NodeDefinition[];
  search(query: string): NodeDefinition[];
}
```

### 5.2 Graph Mutation Events

Every canvas action emits a typed event consumed by `graphStore`:

| Event | Payload |
|---|---|
| `node:add` | `NodeModel` |
| `node:remove` | `{ id: string }` |
| `node:move` | `{ id, position }` |
| `node:config` | `{ id, config }` |
| `edge:add` | `EdgeModel` |
| `edge:remove` | `{ id: string }` |
| `graph:reset` | `WorkflowJSON` |

### 5.3 Topological Sort

```
Algorithm: Kahn's Algorithm (BFS-based)
Complexity: O(V + E)
Cycle detection: throws CyclicGraphError with offending node IDs
```

---

## 6. Code Generation Internals

### 6.1 AST Strategy

N2N uses `recast` to build a JavaScript AST programmatically, then `prettier` to format output.

```
NodeModel → NodeDefinition.generateCode() → ASTNode
                                              ↓
                              ScopeResolver names variables
                                              ↓
                              CodeEmitter assembles final AST
                                              ↓
                              recast.print() → source string
                                              ↓
                              prettier.format() → final file
```

### 6.2 Scope Naming Convention

| Pattern | Example |
|---|---|
| Node output var | `node_{shortId}_out` |
| Error var | `node_{shortId}_err` |
| Loop item var | `node_{shortId}_item` |
| Merged result | `node_{shortId}_merged` |

---

## 7. Runtime Internals

### 7.1 Sandboxed Execution (vm2)

```typescript
const sandbox = new VM({
  timeout: nodeConfig.timeout ?? 30000,
  sandbox: {
    require: safeRequire,    // whitelisted modules only
    console: nodeLogger,
    process: { env: safeEnv },
  },
  eval: false,
  wasm: false,
});
```

### 7.2 Parallel Branch Handling

```
[Split Node] fans out → multiple branches execute via Promise.all()
[Merge Node] awaits  → all branch results collected into array
```

---

## 8. Plugin Loading (ESM Dynamic Import)

```typescript
// Plugins loaded in Main Process
const pluginModule = await import(/* @vite-ignore */ pluginPath);
const plugin: PluginDefinition = pluginModule.default;

// Each node from plugin registered into shared NodeRegistry
plugin.nodes.forEach(node => nodeRegistry.register(node));

// Registry synced to Renderer via IPC on load
ipcMain.handle('plugin:list', () => nodeRegistry.getAll());
```

---

## 9. State Sync: Main ↔ Renderer

```
Renderer graphStore mutation
         ↓
  debounced auto-save (500ms)
         ↓
  IPC: project:save → Main Process
         ↓
  fs.writeFile(projectPath, JSON.stringify(workflowJSON))
```

---

## 10. Error Boundaries

| Layer | Error Type | Handling |
|---|---|---|
| Canvas | Render crash | React ErrorBoundary → show fallback UI |
| Graph Engine | Cyclic graph | `CyclicGraphError` → highlight offending edge |
| Code Gen | Missing config | `ValidationError` → show node config warning |
| Runtime | Node timeout | `TimeoutError` → mark node red, continue or halt |
| Plugin Load | Bad schema | `PluginValidationError` → skip node, log warning |
| File I/O | Permission denied | `FSError` → toast notification |