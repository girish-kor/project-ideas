# DEBUGGING.md — N2N Toolz Debugging System

---

## 1. Debug Modes

| Mode | Description |
|---|---|
| **Live Debug** | Breakpoints + step through nodes during embedded runtime |
| **Execution Log** | Timeline of node statuses, durations, outputs |
| **Error Inspector** | Full stack traces with jump-to-node |
| **Variable Watch** | Monitor expressions during run |
| **Network Inspector** | HTTP request/response log |
| **Replay** | Re-run last execution with same input |

---

## 2. Breakpoints

### Setting Breakpoints

- Right-click node → **Add Breakpoint**
- Click red dot in node header (toggle)
- Keyboard: select node → `B`

### Breakpoint Model

```typescript
interface Breakpoint {
  nodeId:     string;
  enabled:    boolean;
  condition?: string;  // JS expression: e.g. "input.status > 400"
  hitCount:   number;  // auto-incremented
}
```

### Conditional Breakpoints

Available context in condition expression:

| Variable | Description |
|---|---|
| `input` | Node input object |
| `config` | Node config object |
| `env` | Environment variables |
| `runId` | Current run ID |
| `hitCount` | How many times this breakpoint has fired |

**Example conditions:**
```javascript
input.status >= 400
input.data.length === 0
config.url.includes('prod')
hitCount % 5 === 0       // break every 5th hit
```

---

## 3. Debug Interceptor

```typescript
class DebugInterceptor {
  private pausePromise: Promise<void> | null = null;
  private resumeFn: (() => void) | null = null;

  async beforeNode(node: NodeModel, input: NodeInput): Promise<void> {
    const bp = debugStore.getState().breakpoints.get(node.id);
    if (!bp?.enabled) return;

    // Evaluate condition
    if (bp.condition) {
      const hit = await evalCondition(bp.condition, { input, config: node.config });
      if (!hit) return;
    }

    // Pause execution
    debugStore.getState().setPausedAt({ nodeId: node.id, input });
    executionStore.getState().setRunStatus('paused');

    this.pausePromise = new Promise(resolve => {
      this.resumeFn = resolve;
    });

    await this.pausePromise;
    debugStore.getState().setPausedAt(null);
  }

  resume():    void { this.resumeFn?.(); }
  stepOver():  void { this.resumeFn?.(); /* interceptor skips next node bp */ }
  stepInto():  void { /* descends into sub-workflow */ }
}
```

---

## 4. Execution Log

### Log Entry Model

```typescript
interface ExecutionLogEntry {
  id:           string;
  runId:        string;
  nodeId:       string;
  nodeLabel:    string;
  nodeType:     string;
  status:       NodeRunStatus;
  startedAt:    number;          // epoch ms
  completedAt?: number;
  duration?:    number;          // ms
  inputSize?:   number;          // bytes (serialized)
  outputSize?:  number;
  error?:       string;
  retryCount?:  number;
}
```

### Execution Log Tab UI

```
Execution Log                              [Clear] [Filter ▼]
─────────────────────────────────────────────────────────────
✅ HTTP Trigger           node_a1b2   →    2ms
✅ JSON Parse             node_c3d4   →    0ms
✅ MySQL Query            node_e5f6   →  124ms
❌ HTTP Response          node_g7h8   →    3ms   Error: Cannot set headers
─────────────────────────────────────────────────────────────
Total: 4 nodes | Duration: 129ms | 1 error
```

---

## 5. NodeLogger

Injected via `ExecutionContext`. All log calls appear in Console tab.

```typescript
class NodeLogger {
  constructor(private nodeId: string, private runId: string) {}

  debug(msg: string, data?: unknown): void {
    this.emit('debug', msg, data);
  }
  info(msg: string, data?: unknown): void {
    this.emit('info', msg, data);
  }
  warn(msg: string, data?: unknown): void {
    this.emit('warn', msg, data);
  }
  error(msg: string, err?: Error): void {
    this.emit('error', msg, { message: err?.message, stack: err?.stack });
  }

  private emit(level: LogLevel, msg: string, data?: unknown): void {
    const entry: LogEntry = {
      id:        generateId(),
      runId:     this.runId,
      nodeId:    this.nodeId,
      level,
      message:   msg,
      data,
      timestamp: Date.now(),
    };
    executionStore.getState().appendLog(entry);
    ipcMain.emit('runtime:event', { type: 'log', ...entry });
  }
}
```

---

## 6. Console Tab

```
Console                    [Clear] [Level: ALL ▼] [Node: ALL ▼]  [🔍 search]
─────────────────────────────────────────────────────────────────────────────
12:00:01.123  INFO   [HTTP Trigger]   Received POST /api/users
12:00:01.124  DEBUG  [JSON Parse]     Parsing 248 bytes
12:00:01.250  INFO   [MySQL Query]    Query executed: 1 row affected
12:00:01.253  ERROR  [HTTP Response]  Cannot set headers after sent
              Stack: Error: Cannot set headers...
                     at ServerResponse.setHeader (http.js:...)
                     at node_g7h8 (routes/main.js:42)
```

---

## 7. Variable Explorer (Variables Tab)

Shows the full runtime state after each node executes:

```
Variables                                      [📋 Copy] [✏️ Edit]
────────────────────────────────────────────────────────────────────
▼ Run: run_abc123
  ▼ n_a1b2_out     (HTTP Trigger output)
      method:  "POST"
      path:    "/api/users"
    ▼ body:
        name:  "Girish"
        email: "girish@example.com"
  ▼ n_c3d4_out     (JSON Parse output)
      parsed:  { name: "Girish", email: "girish@example.com" }
  ▼ n_e5f6_out     (MySQL Query output)
      rows:          []
      affectedRows:  1
      insertId:      42
```

---

## 8. Network Inspector (Network Tab)

Logs all HTTP calls made by HTTP Request nodes:

```
Network                                     [Clear] [Filter ▼]
────────────────────────────────────────────────────────────────
POST  https://api.example.com/users   200  124ms
  ▼ Request
    Headers: { Authorization: Bearer [REDACTED], Content-Type: application/json }
    Body:    { "name": "Girish" }
  ▼ Response
    Status:  200 OK
    Headers: { content-type: application/json }
    Body:    { "id": 42, "name": "Girish" }
```

---

## 9. Error Inspector

```typescript
interface RuntimeError {
  id:         string;
  runId:      string;
  nodeId:     string;
  nodeLabel:  string;
  nodeType:   string;
  message:    string;
  stack?:     string;
  input?:     unknown;      // input that caused the error
  timestamp:  number;
}
```

**Errors Tab Actions:**
- Click error → jumps to node on canvas (highlights red)
- "Copy Stack" → copies stack trace to clipboard
- "Replay with this input" → replays run with same input that caused error

---

## 10. Replay Last Run

```typescript
class RunReplayer {
  private lastRun: { input: unknown; workflowJSON: WorkflowJSON } | null = null;

  storeRun(input: unknown, workflow: WorkflowJSON): void {
    this.lastRun = { input, workflowJSON: structuredClone(workflow) };
  }

  async replay(): Promise<RunResult> {
    if (!this.lastRun) throw new Error('No previous run to replay');
    const runner = new WorkflowRunner(this.lastRun.workflowJSON);
    return runner.run(this.lastRun.input);
  }
}
```

**Useful for:** Reproducing errors after fixing node config without re-triggering the original event.
