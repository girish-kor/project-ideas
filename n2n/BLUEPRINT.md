# BLUEPRINT.md — N2N Toolz System Architecture & Development Blueprint

> **N2N Toolz** | Visual Node-to-Node Development Platform for Node.js
> Version: 1.0.0 | Last Updated: 2026-03-10

---

## 1. System Overview

N2N Toolz converts a **visual node graph** into **production-ready Node.js source code**. The platform operates in two modes:

| Mode | Description |
|---|---|
| **Design Mode** | User builds workflows visually on a canvas |
| **Runtime Mode** | Workflow executes either in embedded runtime or exported Node.js code |

### High-Level Flow
```
[Visual Canvas] → [Graph JSON] → [AST Builder] → [Code Generator] → [Node.js Source]
                                      ↕
                              [Live Runtime Executor]
```

---

## 2. Core Architecture

### 2.1 Layer Diagram
```
┌─────────────────────────────────────────────────────────┐
│                     UI LAYER                            │
│   Canvas Engine | Node Palette | Properties Panel       │
├─────────────────────────────────────────────────────────┤
│                  GRAPH ENGINE LAYER                     │
│   Node Registry | Edge Manager | Graph Store            │
├─────────────────────────────────────────────────────────┤
│               CODE GENERATION LAYER                     │
│   AST Builder | Template Engine | Code Emitter          │
├─────────────────────────────────────────────────────────┤
│                 RUNTIME / EXECUTOR LAYER                │
│   Workflow Runner | Sandbox VM | Debug Interceptor      │
├─────────────────────────────────────────────────────────┤
│                    PLUGIN LAYER                         │
│   Plugin Loader | Custom Node Registry | Validator      │
├─────────────────────────────────────────────────────────┤
│                  PERSISTENCE LAYER                      │
│   Project FS | State Store | Undo History               │
└─────────────────────────────────────────────────────────┘
```

### 2.2 Technology Stack

| Layer | Technology |
|---|---|
| Desktop App Shell | Electron + Node.js |
| UI Framework | React 18 + TypeScript |
| Canvas Engine | React Flow (or custom WebGL canvas) |
| State Management | Zustand + Immer |
| Code Generation | Custom AST + `recast` / `escodegen` |
| Runtime Sandbox | `vm2` or Node.js `vm` module |
| Plugin System | ESM dynamic imports |
| File I/O | Node.js `fs/promises` |
| Build Tooling | Vite + esbuild |

---

## 3. Visual Node Engine

### 3.1 Canvas Responsibilities
- Render nodes and edges via React Flow
- Handle drag/drop from Node Palette to canvas
- Emit graph mutation events to Graph Store
- Manage viewport state (zoom, pan, selection)

### 3.2 Node Visual Anatomy

```
┌──────────────────────────────┐
│  [●] HTTP Request            │  ← Node header (type icon + label)
├──────────────────────────────┤
│  ○ trigger   response ○      │  ← Input / Output ports
│  ○ headers                   │
├──────────────────────────────┤
│  URL: [________________]     │  ← Inline config fields (optional)
│  Method: [GET ▼]             │
└──────────────────────────────┘
```

### 3.3 Port Types

| Port Type | Color Code | Description |
|---|---|---|
| `any` | Gray | Accepts any data type |
| `object` | Blue | JSON Object |
| `array` | Purple | Array payload |
| `string` | Green | String value |
| `number` | Orange | Numeric value |
| `boolean` | Yellow | Boolean flag |
| `stream` | Cyan | Node.js Readable stream |
| `error` | Red | Error object port |

### 3.4 Edge Validation Rules
- Ports of mismatched types show warning (not blocked)
- `error` ports can only connect to error-handler nodes
- Loop-back edges flagged as circular (blocked unless inside Loop node)

---

## 4. Node Execution Model

### 4.1 Execution Modes

| Mode | Description |
|---|---|
| **Sequential** | Nodes execute in topological order, one at a time |
| **Parallel** | Split node fans out to multiple branches concurrently |
| **Streaming** | Data flows chunk-by-chunk through stream-aware nodes |
| **Event-Driven** | Trigger nodes listen for events and fire downstream |

### 4.2 Node Lifecycle

```
INIT → READY → EXECUTING → [SUCCESS | ERROR] → COMPLETE
                                ↓
                         RETRY (if configured)
```

### 4.3 Execution Context Object (passed between nodes)

```typescript
interface ExecutionContext {
  nodeId: string;
  workflowId: string;
  runId: string;
  input: Record<string, unknown>;
  output: Record<string, unknown>;
  env: Record<string, string>;
  logger: NodeLogger;
  state: WorkflowState;
  abort: AbortController;
}
```

### 4.4 Topological Sort Algorithm

```
function topoSort(graph: Graph): NodeID[] {
  visited = new Set()
  order = []
  for each node in graph.nodes:
    if not visited: DFS(node)
  return order.reverse()
}
```

---

## 5. Node Graph Data Structure

### 5.1 Core Graph Schema (JSON)

```json
{
  "id": "wf_uuid_v4",
  "name": "My Workflow",
  "version": "1.0.0",
  "nodes": [
    {
      "id": "node_001",
      "type": "http-trigger",
      "label": "API Entry",
      "position": { "x": 100, "y": 200 },
      "config": {
        "method": "POST",
        "path": "/api/data"
      },
      "ports": {
        "outputs": ["response", "headers"]
      }
    },
    {
      "id": "node_002",
      "type": "json-parse",
      "label": "Parse Body",
      "position": { "x": 350, "y": 200 },
      "config": {},
      "ports": {
        "inputs": ["raw"],
        "outputs": ["parsed", "error"]
      }
    }
  ],
  "edges": [
    {
      "id": "edge_001",
      "source": "node_001",
      "sourcePort": "response",
      "target": "node_002",
      "targetPort": "raw"
    }
  ],
  "metadata": {
    "created": "2026-03-10T00:00:00Z",
    "author": "GIRISH",
    "tags": ["api", "json"]
  }
}
```

### 5.2 In-Memory Graph Store (Zustand slice)

```typescript
interface GraphStore {
  nodes: Map<string, NodeModel>;
  edges: Map<string, EdgeModel>;
  addNode: (node: NodeModel) => void;
  removeNode: (id: string) => void;
  updateNodeConfig: (id: string, config: Partial<NodeConfig>) => void;
  addEdge: (edge: EdgeModel) => void;
  removeEdge: (id: string) => void;
  getAdjacentNodes: (id: string) => NodeModel[];
  serialize: () => WorkflowJSON;
  deserialize: (json: WorkflowJSON) => void;
}
```

---

## 6. Code Generation Engine

### 6.1 Pipeline

```
Graph JSON
   ↓
[Graph Analyzer]        → validates, resolves types, builds dependency map
   ↓
[AST Builder]           → constructs JavaScript AST nodes per graph node
   ↓
[Scope Resolver]        → maps data flow between nodes to variable names
   ↓
[Code Emitter]          → serializes AST → JS source string
   ↓
[Formatter]             → runs Prettier on output
   ↓
Generated Node.js File(s)
```

### 6.2 Code Generation Strategy Per Node Type

| Node Type | Generated Code Pattern |
|---|---|
| HTTP Trigger | `express.Router()` handler function |
| If/Else | `if (condition) { ... } else { ... }` |
| Loop (forEach) | `for (const item of collection) { ... }` |
| HTTP Request | `await axios.get/post(url, options)` |
| Function Node | Inline IIFE or named async function |
| Database Query | `await db.query(sql, params)` |
| Try/Catch | `try { ... } catch(err) { ... }` |
| Merge | `Promise.all([...branches])` |
| Delay | `await new Promise(r => setTimeout(r, ms))` |

### 6.3 Example: HTTP Trigger → JSON Parse → Respond

**Visual Graph:**
```
[HTTP Trigger] ──► [JSON Parse] ──► [HTTP Response]
```

**Generated Code:**
```javascript
// Generated by N2N Toolz — DO NOT EDIT MANUALLY
import express from 'express';
const router = express.Router();

router.post('/api/data', async (req, res) => {
  try {
    // node_001: HTTP Trigger
    const node_001_output = req.body;

    // node_002: JSON Parse
    const node_002_output = typeof node_001_output === 'string'
      ? JSON.parse(node_001_output)
      : node_001_output;

    // node_003: HTTP Response
    res.status(200).json(node_002_output);
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

export default router;
```

### 6.4 AST Node Map

```typescript
interface ASTNodeBuilder {
  buildFunction(node: NodeModel): FunctionDeclaration;
  buildVariableDecl(name: string, init: Expression): VariableDeclaration;
  buildAwaitExpr(callee: Expression, args: Expression[]): AwaitExpression;
  buildIfStmt(test: Expression, consequent: BlockStatement, alternate?: BlockStatement): IfStatement;
  buildTryCatch(block: BlockStatement, handler: CatchClause): TryStatement;
}
```

---

## 7. Plugin System for Custom Nodes

### 7.1 Plugin Package Structure

```
my-n2n-plugin/
├── package.json              → { "n2n": { "type": "node-plugin" } }
├── index.js                  → Plugin entry point (exports node definitions)
├── nodes/
│   ├── MyCustomNode.js       → Node logic + codegen template
│   └── AnotherNode.js
├── icons/
│   └── custom-icon.svg
└── README.md
```

### 7.2 Plugin Entry Point Contract

```javascript
// index.js
export default {
  name: 'my-plugin',
  version: '1.0.0',
  nodes: [MyCustomNode, AnotherNode],
};
```

### 7.3 Custom Node Definition Contract

```javascript
export const MyCustomNode = {
  type: 'my-custom-node',
  category: 'Integrations',
  label: 'My Custom Node',
  icon: './icons/custom-icon.svg',
  inputs: [{ name: 'data', type: 'object' }],
  outputs: [{ name: 'result', type: 'object' }, { name: 'error', type: 'error' }],
  configSchema: {
    apiKey: { type: 'string', label: 'API Key', required: true, secret: true },
    endpoint: { type: 'string', label: 'Endpoint URL', default: 'https://api.example.com' },
  },
  execute: async (input, config, ctx) => {
    // Live execution logic (used in embedded runtime)
    const result = await fetch(config.endpoint, {
      headers: { Authorization: `Bearer ${config.apiKey}` },
      body: JSON.stringify(input.data),
    });
    return { result: await result.json() };
  },
  generateCode: (node, scope) => {
    // Returns AST-compatible code string for this node
    return `
      const ${scope.outputVar} = await myCustomFetch(
        '${node.config.endpoint}',
        '${node.config.apiKey}',
        ${scope.inputVar}
      );
    `;
  },
};
```

### 7.4 Plugin Loader

```typescript
class PluginLoader {
  registry: Map<string, NodeDefinition> = new Map();

  async load(pluginPath: string): Promise<void> {
    const plugin = await import(pluginPath);
    for (const nodeDef of plugin.default.nodes) {
      this.validate(nodeDef);
      this.registry.set(nodeDef.type, nodeDef);
    }
  }

  validate(def: NodeDefinition): void {
    // Checks: type, execute fn, generateCode fn, ports
  }

  getNode(type: string): NodeDefinition {
    return this.registry.get(type)!;
  }
}
```

---

## 8. Workflow Runtime

### 8.1 Runtime Modes

| Mode | Description |
|---|---|
| **Embedded** | Runs inside app process via `vm2` sandbox |
| **Dev Server** | Spawns child Node.js process, hot-reloads on save |
| **Exported** | Standalone `node dist/index.js` |

### 8.2 Workflow Runner Algorithm

```typescript
class WorkflowRunner {
  async run(graph: Graph, input: unknown): Promise<RunResult> {
    const order = topoSort(graph);
    const outputs: Map<string, unknown> = new Map();

    for (const nodeId of order) {
      const node = graph.nodes.get(nodeId);
      const nodeInput = this.resolveInputs(node, outputs, graph.edges);
      const ctx = this.buildContext(node, input);

      try {
        const result = await this.executeNode(node, nodeInput, ctx);
        outputs.set(nodeId, result);
        this.emit('node:success', { nodeId, result });
      } catch (err) {
        outputs.set(nodeId, { __error: err });
        this.emit('node:error', { nodeId, err });
        if (!node.config.continueOnError) throw err;
      }
    }
    return { outputs, status: 'success' };
  }
}
```

### 8.3 Input Resolution

```typescript
resolveInputs(node: NodeModel, outputs: Map, edges: Edge[]): Record<string, unknown> {
  const incomingEdges = edges.filter(e => e.target === node.id);
  return incomingEdges.reduce((acc, edge) => {
    acc[edge.targetPort] = outputs.get(edge.source)?.[edge.sourcePort];
    return acc;
  }, {});
}
```

---

## 9. State Management

### 9.1 Store Slices (Zustand)

| Store | Responsibility |
|---|---|
| `graphStore` | Nodes, edges, serialization |
| `uiStore` | Panel visibility, zoom, selection, theme |
| `executionStore` | Run state, node statuses, outputs, errors |
| `projectStore` | Project meta, open workflows, recent files |
| `pluginStore` | Loaded plugins, node registry |
| `historyStore` | Undo/redo stack (snapshots of graphStore) |

### 9.2 History (Undo/Redo) Implementation

```typescript
interface HistoryStore {
  past: GraphSnapshot[];
  future: GraphSnapshot[];
  push: (snapshot: GraphSnapshot) => void;
  undo: () => void;
  redo: () => void;
}
// Snapshot = serialized graphStore state (nodes + edges as plain JSON)
// Max stack depth: 100
```

---

## 10. UI Architecture

### 10.1 Component Tree

```
<App>
├── <TitleBar />
├── <TopMenuBar />
├── <MainLayout>
│   ├── <NodePalette />          ← Left panel
│   ├── <CanvasArea>
│   │   ├── <WorkflowTabBar />
│   │   ├── <ReactFlowCanvas>
│   │   │   ├── <NodeRenderer />    (per node)
│   │   │   ├── <EdgeRenderer />    (per edge)
│   │   │   └── <SelectionBox />
│   │   └── <CanvasToolbar />    ← zoom, fit, grid controls
│   └── <PropertiesPanel />      ← Right panel
└── <BottomPanel>
    ├── <ConsoleTab />
    ├── <ExecutionLogTab />
    ├── <VariablesTab />
    └── <ErrorsTab />
```

### 10.2 Custom Node Renderer Contract

```typescript
interface NodeRendererProps {
  id: string;
  data: NodeModel;
  selected: boolean;
  onConfigChange: (config: Partial<NodeConfig>) => void;
}
// Each node type registers its own renderer component via NodeRegistry
```

---

## 11. File / Project Structure

### 11.1 Application Source Tree

```
n2n-toolz/
├── electron/
│   ├── main.ts                 → Electron main process
│   └── preload.ts              → IPC bridge
├── src/
│   ├── app/
│   │   ├── App.tsx
│   │   └── routes.tsx
│   ├── components/
│   │   ├── canvas/
│   │   │   ├── CanvasArea.tsx
│   │   │   ├── NodeRenderer.tsx
│   │   │   └── EdgeRenderer.tsx
│   │   ├── panels/
│   │   │   ├── NodePalette.tsx
│   │   │   ├── PropertiesPanel.tsx
│   │   │   └── BottomPanel.tsx
│   │   └── menus/
│   │       └── TopMenuBar.tsx
│   ├── engine/
│   │   ├── graph/
│   │   │   ├── GraphStore.ts
│   │   │   ├── GraphSerializer.ts
│   │   │   └── topoSort.ts
│   │   ├── codegen/
│   │   │   ├── ASTBuilder.ts
│   │   │   ├── ScopeResolver.ts
│   │   │   ├── CodeEmitter.ts
│   │   │   └── templates/
│   │   │       ├── httpTrigger.ts
│   │   │       ├── ifElse.ts
│   │   │       └── ...
│   │   ├── runtime/
│   │   │   ├── WorkflowRunner.ts
│   │   │   ├── NodeExecutor.ts
│   │   │   └── Sandbox.ts
│   │   └── plugins/
│   │       ├── PluginLoader.ts
│   │       └── NodeRegistry.ts
│   ├── nodes/
│   │   ├── core/
│   │   │   ├── HttpTriggerNode.ts
│   │   │   ├── IfElseNode.ts
│   │   │   ├── LoopNode.ts
│   │   │   └── ...
│   │   └── index.ts            → registers all core nodes
│   ├── stores/
│   │   ├── graphStore.ts
│   │   ├── uiStore.ts
│   │   ├── executionStore.ts
│   │   └── historyStore.ts
│   ├── types/
│   │   ├── graph.types.ts
│   │   ├── node.types.ts
│   │   └── codegen.types.ts
│   └── utils/
│       ├── idGenerator.ts
│       └── portValidator.ts
├── plugins/                    → installed plugin packages
├── projects/                   → user project files (.n2n)
├── dist/                       → generated Node.js output
├── package.json
├── vite.config.ts
└── tsconfig.json
```

### 11.2 Project File Format (.n2n)

```
my-project/
├── project.json               → project metadata
├── workflows/
│   ├── main.n2nflow           → main workflow graph JSON
│   └── helpers.n2nflow
├── env/
│   └── .env.dev               → environment variables
└── assets/
    └── ...
```

---

## 12. Node Definition Schema

### 12.1 Full Node Schema (TypeScript)

```typescript
interface NodeDefinition {
  // Identity
  type: string;                         // Unique type key e.g. 'http-request'
  version: string;                      // semver
  category: NodeCategory;               // 'Core' | 'HTTP' | 'Database' | ...
  label: string;                        // Display name
  description: string;
  icon: string;                         // SVG path or URL

  // Ports
  inputs: PortDefinition[];
  outputs: PortDefinition[];

  // Configuration schema (drives Properties Panel form rendering)
  configSchema: Record<string, FieldSchema>;

  // Execution
  execute: (input: unknown, config: NodeConfig, ctx: ExecutionContext) => Promise<unknown>;

  // Code generation
  generateCode: (node: NodeModel, scope: ScopeMap) => string | ASTNode;

  // Optional
  onInit?: (config: NodeConfig) => void;
  validate?: (config: NodeConfig) => ValidationResult;
  testable?: boolean;
}

interface PortDefinition {
  name: string;
  type: DataType;
  required?: boolean;
  multiple?: boolean;         // accepts multiple incoming connections
  description?: string;
}

interface FieldSchema {
  type: 'string' | 'number' | 'boolean' | 'select' | 'json' | 'code' | 'secret';
  label: string;
  default?: unknown;
  required?: boolean;
  options?: string[];         // for 'select' type
  placeholder?: string;
  validation?: RegExp | string;
}
```

---

## 13. API Layer

### 13.1 IPC API (Electron Main ↔ Renderer)

| Channel | Direction | Payload | Response |
|---|---|---|---|
| `project:open` | Renderer → Main | `{ path: string }` | `WorkflowJSON` |
| `project:save` | Renderer → Main | `{ path, data: WorkflowJSON }` | `{ success: boolean }` |
| `codegen:generate` | Renderer → Main | `WorkflowJSON` | `{ files: GeneratedFile[] }` |
| `runtime:run` | Renderer → Main | `{ workflowId, input }` | `RunResult` (streamed) |
| `runtime:stop` | Renderer → Main | `{ runId }` | `void` |
| `plugin:install` | Renderer → Main | `{ source: string }` | `PluginMeta` |
| `plugin:list` | Renderer → Main | `{}` | `PluginMeta[]` |

### 13.2 REST API (Dev Server Mode — optional)

```
POST   /api/workflow/run          → Trigger workflow execution
GET    /api/workflow/:id/status   → Get run status
GET    /api/workflow/:id/output   → Get run output
POST   /api/code/generate         → Generate code from graph JSON
GET    /api/plugins               → List installed plugins
POST   /api/plugins/install       → Install plugin from registry
```

---

## 14. Debugging and Logging System

### 14.1 Breakpoint Model

```typescript
interface Breakpoint {
  nodeId: string;
  condition?: string;    // optional JS expression evaluated at runtime
  enabled: boolean;
}
```

### 14.2 Debug Interceptor

```typescript
class DebugInterceptor {
  breakpoints: Map<string, Breakpoint>;

  async before(node: NodeModel, input: unknown): Promise<void> {
    const bp = this.breakpoints.get(node.id);
    if (bp?.enabled) {
      if (!bp.condition || eval(bp.condition)) {
        await this.pauseExecution(node.id, input);  // suspends runner
      }
    }
  }
}
```

### 14.3 Logging Levels

| Level | Color | Used For |
|---|---|---|
| `DEBUG` | Gray | Internal engine messages |
| `INFO` | White | Normal execution messages |
| `WARN` | Yellow | Non-fatal issues |
| `ERROR` | Red | Node or runtime failures |
| `TRACE` | Cyan | Data flow between nodes |

### 14.4 NodeLogger (per-node logger injected via context)

```typescript
interface NodeLogger {
  debug(msg: string, data?: unknown): void;
  info(msg: string, data?: unknown): void;
  warn(msg: string, data?: unknown): void;
  error(msg: string, err?: Error): void;
}
```

---

## 15. Export System

### 15.1 Export Targets

| Target | Description | Output |
|---|---|---|
| **Node.js (ESM)** | ES Modules source | `dist/index.js` + `dist/routes/` |
| **Node.js (CJS)** | CommonJS source | `dist/index.cjs` |
| **TypeScript** | Typed .ts files | `dist/src/` |
| **Docker** | App + Dockerfile | `dist/` + `Dockerfile` |
| **ZIP Bundle** | All of above zipped | `project.zip` |
| **JSON Graph** | Raw workflow JSON | `workflow.n2nflow` |

### 15.2 Generated Project Structure (Node.js Export)

```
dist/
├── index.js                  → App entry point (express or standalone)
├── routes/
│   └── [workflow-name].js    → HTTP trigger routes
├── workflows/
│   └── [workflow-name].js    → Workflow functions (non-HTTP)
├── lib/
│   └── helpers.js            → Shared utility functions
├── package.json              → Dependencies inferred from used nodes
└── .env.example              → Environment variable template
```

### 15.3 Export Pipeline Implementation

```typescript
class ExportService {
  async exportNodeJS(graph: Graph, options: ExportOptions): Promise<ExportResult> {
    const analyzed = await this.analyzer.analyze(graph);
    const ast = await this.astBuilder.build(analyzed);
    const code = await this.emitter.emit(ast);
    const formatted = await prettier.format(code, { parser: 'babel' });
    const deps = this.dependencyResolver.resolve(graph);
    const packageJson = this.generatePackageJson(deps);
    return {
      files: [
        { path: 'dist/index.js', content: formatted },
        { path: 'package.json', content: JSON.stringify(packageJson, null, 2) },
      ]
    };
  }
}
```

### 15.4 Dependency Resolution

```typescript
// Maps node types to npm packages
const NODE_DEPENDENCIES: Record<string, string[]> = {
  'http-request':   ['axios'],
  'http-trigger':   ['express'],
  'mysql-query':    ['mysql2'],
  'postgres-query': ['pg'],
  'mongodb-query':  ['mongoose'],
  'redis-get':      ['ioredis'],
  'smtp-email':     ['nodemailer'],
  'jwt-sign':       ['jsonwebtoken'],
  'hash':           ['bcryptjs'],
};
```

---

## 16. Scalability Considerations

### 16.1 Performance Targets

| Metric | Target |
|---|---|
| Canvas render (1000 nodes) | < 16ms per frame |
| Graph serialization | < 50ms |
| Code generation (50-node graph) | < 200ms |
| Node execution (per node) | < 5ms overhead |
| Plugin load time | < 300ms per plugin |

### 16.2 Canvas Virtualization
- Only render nodes within viewport bounds
- React Flow `nodeTypes` lazy loading
- Web Worker for topoSort on large graphs (> 500 nodes)

### 16.3 Code Generation Optimization
- Cache AST output per unchanged node
- Incremental regeneration (only rebuild changed subtrees)
- Parallel codegen for independent subgraphs

### 16.4 Runtime Isolation
- Each workflow run gets an isolated `vm.Context`
- Memory limit enforcement via `--max-old-space-size` on child process
- Timeout enforced via `AbortController` with `AbortSignal.timeout(ms)`

### 16.5 Plugin Architecture Limits

| Constraint | Value |
|---|---|
| Max plugins loaded simultaneously | 50 |
| Max nodes per plugin | 100 |
| Max execution timeout per node | 30,000 ms (configurable) |
| Max graph nodes | 10,000 (beyond: paginated sub-workflows) |

### 16.6 Sub-Workflow Composition
- Large workflows broken into Sub-Workflow nodes
- Each sub-workflow generates an independent async function
- Enables recursive workflow composition

```
[Main Workflow]
   └── [Sub-Workflow Node A] → calls generated asyncFunctionA()
   └── [Sub-Workflow Node B] → calls generated asyncFunctionB()
```

---

*End of BLUEPRINT.md*