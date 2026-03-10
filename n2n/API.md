# API.md — N2N Toolz API Reference

---

## 1. IPC API (Electron Main ↔ Renderer)

All IPC calls use `ipcRenderer.invoke(channel, payload)` from Renderer and `ipcMain.handle(channel, handler)` in Main.

### 1.1 Project API

| Channel | Payload | Response | Description |
|---|---|---|---|
| `project:new` | `{ name, path, template? }` | `ProjectMeta` | Create new project |
| `project:open` | `{ path: string }` | `{ meta: ProjectMeta, workflows: WorkflowJSON[] }` | Open .n2n project |
| `project:save` | `{ path, meta, workflows }` | `{ success: boolean }` | Save all workflows |
| `project:saveOne` | `{ path, workflowId, data: WorkflowJSON }` | `{ success: boolean }` | Save single workflow |
| `project:close` | `{ path }` | `void` | Close project |
| `project:delete` | `{ path }` | `{ success: boolean }` | Delete project directory |
| `project:list-recent` | `{}` | `RecentProject[]` | Get last 10 opened projects |
| `project:settings` | `{ path }` | `ProjectSettings` | Read project settings |
| `project:settings:save` | `{ path, settings }` | `{ success: boolean }` | Write project settings |

---

### 1.2 Code Generation API

| Channel | Payload | Response | Description |
|---|---|---|---|
| `codegen:generate` | `{ workflow: WorkflowJSON, options: CodeGenOptions }` | `GeneratedFile[]` | Generate Node.js source files |
| `codegen:preview` | `{ workflow: WorkflowJSON, nodeId?: string }` | `string` | Preview generated code (no write) |
| `codegen:validate` | `{ workflow: WorkflowJSON }` | `GraphValidationResult` | Validate before generating |
| `codegen:dependencies` | `{ workflow: WorkflowJSON }` | `Record<string, string>` | List npm deps needed |

```typescript
interface CodeGenOptions {
  moduleSystem:   'esm' | 'cjs';
  typescript:     boolean;
  outputDir:      string;
  prettier:       boolean;
  comments:       boolean;       // include source comments
  envFile:        boolean;       // generate .env.example
}

interface GeneratedFile {
  path:     string;              // relative to outputDir
  content:  string;
  type:     'route' | 'workflow' | 'helper' | 'config' | 'package';
}
```

---

### 1.3 Runtime API

| Channel | Payload | Response | Description |
|---|---|---|---|
| `runtime:run` | `{ workflowId, input?, options? }` | `RunHandle` (streaming) | Start workflow execution |
| `runtime:stop` | `{ runId }` | `void` | Abort active run |
| `runtime:pause` | `{ runId }` | `void` | Pause at next breakpoint |
| `runtime:resume` | `{ runId }` | `void` | Resume paused run |
| `runtime:step` | `{ runId }` | `void` | Step to next node |
| `runtime:getOutput` | `{ runId, nodeId }` | `NodeOutput` | Get specific node output |
| `runtime:history` | `{ workflowId, limit? }` | `RunRecord[]` | Get past run records |

```typescript
// Streaming: runtime uses event subscription for progress
ipcRenderer.on('runtime:event', (_, event: RuntimeEvent) => { ... });

interface RuntimeEvent {
  runId:   string;
  type:    'node:start' | 'node:success' | 'node:error' | 'node:skip' | 'run:complete' | 'log';
  nodeId?: string;
  data?:   unknown;
}
```

---

### 1.4 Plugin API

| Channel | Payload | Response | Description |
|---|---|---|---|
| `plugin:install` | `{ source: string }` | `PluginMeta` | Install from npm/path/github |
| `plugin:uninstall` | `{ name: string }` | `{ success: boolean }` | Remove plugin |
| `plugin:list` | `{}` | `PluginMeta[]` | List installed plugins |
| `plugin:reload` | `{ name?: string }` | `PluginMeta[]` | Reload one or all plugins |
| `plugin:search` | `{ query: string }` | `RegistryResult[]` | Search plugin registry |
| `plugin:update` | `{ name: string }` | `PluginMeta` | Update to latest version |

```typescript
interface PluginMeta {
  name:        string;
  version:     string;
  description: string;
  author:      string;
  nodeCount:   number;
  nodeTypes:   string[];
  installed:   boolean;
  enabled:     boolean;
  path:        string;
}
```

---

### 1.5 File System API

| Channel | Payload | Response |
|---|---|---|
| `fs:read` | `{ path }` | `string` |
| `fs:write` | `{ path, content }` | `{ success: boolean }` |
| `fs:exists` | `{ path }` | `boolean` |
| `fs:list` | `{ dir }` | `FileEntry[]` |
| `fs:mkdir` | `{ path }` | `{ success: boolean }` |
| `fs:delete` | `{ path }` | `{ success: boolean }` |
| `fs:pick` | `{ mode: 'open' | 'save', filters }` | `string | null` |

---

### 1.6 Deployment API

| Channel | Payload | Response |
|---|---|---|
| `deploy:build` | `{ projectPath, options }` | `BuildResult` |
| `deploy:startDevServer` | `{ distPath, port }` | `{ pid: number, url: string }` |
| `deploy:stopDevServer` | `{ pid }` | `void` |
| `deploy:ssh` | `{ target: SSHTarget, distPath }` | `DeployResult` (streaming) |
| `deploy:docker` | `{ distPath, tag }` | `DeployResult` (streaming) |
| `deploy:targets:list` | `{}` | `DeployTarget[]` |
| `deploy:targets:save` | `{ target: DeployTarget }` | `DeployTarget` |

---

## 2. REST API (Dev Server Mode)

When "Start Dev Server" is active, the generated Node.js app exposes:

### 2.1 Workflow Control Endpoints

```
POST   /n2n/run/:workflowId
       Body: { input?: any }
       Response: { runId, status, outputs }

GET    /n2n/status/:runId
       Response: { runId, status, nodeStatuses: NodeRunStatus[] }

DELETE /n2n/run/:runId
       Response: { aborted: boolean }

GET    /n2n/runs?workflowId=&limit=
       Response: RunRecord[]
```

### 2.2 Node Inspection Endpoints

```
GET    /n2n/nodes
       Response: NodeDefinition[] (registered types only, no execute fn)

GET    /n2n/nodes/:type
       Response: NodeDefinition

GET    /n2n/output/:runId/:nodeId
       Response: NodeOutput
```

### 2.3 Health

```
GET    /n2n/health
       Response: { status: 'ok', version, uptime }
```

---

## 3. Plugin Registry API

Base URL: `https://registry.n2ntoolz.io/api/v1`

| Endpoint | Method | Description |
|---|---|---|
| `/plugins` | GET | List all available plugins |
| `/plugins/search?q=` | GET | Search plugins by keyword |
| `/plugins/:name` | GET | Get plugin metadata |
| `/plugins/:name/versions` | GET | List all versions |
| `/plugins/submit` | POST | Submit new plugin to registry |

```typescript
interface RegistryResult {
  name:        string;
  version:     string;
  description: string;
  author:      string;
  downloads:   number;
  rating:      number;
  tags:        string[];
  npmUrl:      string;
}
```

---

## 4. Error Codes

| Code | Name | Description |
|---|---|---|
| `E001` | `GRAPH_INVALID` | Graph failed validation |
| `E002` | `CYCLIC_GRAPH` | Graph contains cycle |
| `E003` | `NODE_NOT_FOUND` | Node type not in registry |
| `E004` | `CODEGEN_FAILED` | Code generation error |
| `E005` | `RUNTIME_TIMEOUT` | Node execution timed out |
| `E006` | `PLUGIN_INVALID` | Plugin schema validation failed |
| `E007` | `PLUGIN_CONFLICT` | Duplicate node type from plugin |
| `E008` | `FS_PERMISSION` | File system permission denied |
| `E009` | `DEPLOY_FAILED` | Deployment pipeline error |
| `E010` | `SANDBOX_VIOLATION` | Code attempted restricted operation |