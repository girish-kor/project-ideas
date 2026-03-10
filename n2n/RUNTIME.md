# RUNTIME.md — N2N Toolz Workflow Execution Runtime

---

## 1. Runtime Modes

| Mode | Trigger | Execution Environment | Use Case |
|---|---|---|---|
| **Embedded** | F5 / Run button in UI | `vm2` sandbox in Main Process | Live testing inside app |
| **Dev Server** | Deploy → Start Dev Server | Child Node.js process | Local development with hot-reload |
| **Exported** | `node dist/index.js` | Native Node.js process | Production deployment |

---

## 2. WorkflowRunner

```typescript
class WorkflowRunner extends EventEmitter {
  private graph: Graph;
  private runId: string;
  private outputs: Map<string, NodeOutput> = new Map();
  private abortControllers: Map<string, AbortController> = new Map();
  private status: RunStatus = 'idle';

  async run(input?: unknown): Promise<RunResult> {
    this.runId = generateUUID();
    this.status = 'running';
    this.emit('run:start', { runId: this.runId });

    const order = topologicalSort(
      Array.from(this.graph.nodes.values()),
      Array.from(this.graph.edges.values())
    );

    for (const nodeId of order) {
      const node = this.graph.nodes.get(nodeId)!;

      if (!node.enabled) {
        this.emit('node:skip', { nodeId });
        continue;
      }

      await this.runNode(node);

      if (this.status === 'aborted') break;
    }

    this.status = 'complete';
    this.emit('run:complete', { runId: this.runId, outputs: this.outputs });
    return { runId: this.runId, outputs: this.outputs, status: this.status };
  }

  private async runNode(node: NodeModel): Promise<void> {
    const nodeInput = this.resolveInputs(node);
    const ctx = this.buildContext(node);
    const def = NodeRegistry.getInstance().get(node.type);

    this.emit('node:start', { nodeId: node.id });

    let attempts = 0;
    const maxAttempts = (node.executionOptions.retryCount ?? 0) + 1;

    while (attempts < maxAttempts) {
      try {
        const output = await def.execute(nodeInput, node.config, ctx);
        this.outputs.set(node.id, output);
        this.emit('node:success', { nodeId: node.id, output });
        return;
      } catch (err) {
        attempts++;
        if (attempts < maxAttempts) {
          await sleep(node.executionOptions.retryDelay ?? 1000);
          this.emit('node:retry', { nodeId: node.id, attempt: attempts, err });
        } else {
          this.outputs.set(node.id, { __error: err });
          this.emit('node:error', { nodeId: node.id, err });
          if (!node.executionOptions.continueOnError) throw err;
        }
      }
    }
  }

  abort(): void {
    this.status = 'aborted';
    for (const ac of this.abortControllers.values()) ac.abort();
    this.emit('run:abort', { runId: this.runId });
  }
}
```

---

## 3. Input Resolution

```typescript
private resolveInputs(node: NodeModel): NodeInput {
  const incoming = Array.from(this.graph.edges.values())
    .filter(e => e.target === node.id);

  return incoming.reduce((acc, edge) => {
    const sourceOutput = this.outputs.get(edge.source);
    if (sourceOutput) {
      acc[edge.targetPort] = (sourceOutput as Record<string, unknown>)[edge.sourcePort];
    }
    return acc;
  }, {} as NodeInput);
}
```

---

## 4. Execution Context Builder

```typescript
private buildContext(node: NodeModel): ExecutionContext {
  const ac = new AbortController();
  this.abortControllers.set(node.id, ac);

  // Apply per-node timeout
  if (node.executionOptions.timeout > 0) {
    setTimeout(() => ac.abort(), node.executionOptions.timeout);
  }

  return {
    nodeId:     node.id,
    workflowId: this.graph.id,
    runId:      this.runId,
    env:        this.safeEnv,
    logger:     new NodeLogger(node.id, this.runId),
    state:      new WorkflowStateAccessor(this.runId),
    abort:      ac,
  };
}
```

---

## 5. Parallel Execution (Split / Merge)

```typescript
// Split node fans out — branches execute concurrently
async runParallelBranches(branches: NodeModel[][]): Promise<unknown[]> {
  return Promise.all(
    branches.map(branch => this.runBranch(branch))
  );
}

// Merge node strategies
type MergeStrategy = 'all' | 'race' | 'allSettled';

async mergeBranches(strategy: MergeStrategy, promises: Promise<unknown>[]): Promise<unknown> {
  switch (strategy) {
    case 'all':        return Promise.all(promises);
    case 'race':       return Promise.race(promises);
    case 'allSettled': return Promise.allSettled(promises);
  }
}
```

---

## 6. Sandbox (Embedded Mode)

```typescript
import { VM } from 'vm2';

class NodeSandbox {
  run(code: string, context: SandboxContext): unknown {
    const vm = new VM({
      timeout:  context.timeout ?? 30000,
      sandbox: {
        input:   context.input,
        config:  context.config,
        env:     context.env,
        logger:  context.logger,
        fetch:   context.allowFetch ? fetch : undefined,
        require: createSafeRequire(WHITELIST_MODULES),
      },
      eval:  false,
      wasm:  false,
    });

    return vm.run(`(async () => { ${code} })()`);
  }
}

// Whitelisted built-in modules for sandbox
const WHITELIST_MODULES = [
  'path', 'url', 'crypto', 'util', 'querystring',
  'stream', 'buffer', 'events', 'assert',
];
```

---

## 7. Dev Server Mode

```typescript
// Spawns a child process running generated Node.js code
class DevServerManager {
  private child: ChildProcess | null = null;

  async start(distPath: string, port: number): Promise<void> {
    this.child = spawn('node', ['index.js'], {
      cwd: distPath,
      env: { ...process.env, PORT: String(port) },
      stdio: ['pipe', 'pipe', 'pipe'],
    });

    this.child.stdout?.on('data', d => this.emitLog('info', d.toString()));
    this.child.stderr?.on('data', d => this.emitLog('error', d.toString()));
    this.child.on('exit', code => this.emit('server:exit', { code }));
  }

  async restart(distPath: string): Promise<void> {
    await this.stop();
    await this.start(distPath, this.port);
  }

  stop(): Promise<void> {
    return new Promise(resolve => {
      if (!this.child) return resolve();
      this.child.kill('SIGTERM');
      this.child.on('exit', () => resolve());
    });
  }
}
```

---

## 8. Run Status Model

```typescript
type RunStatus = 'idle' | 'running' | 'paused' | 'complete' | 'error' | 'aborted';

interface NodeRunStatus {
  nodeId:    string;
  status:    'pending' | 'running' | 'success' | 'error' | 'skipped' | 'retry';
  startedAt?:   number;   // epoch ms
  completedAt?: number;
  duration?:    number;   // ms
  output?:      unknown;
  error?:       string;
  retryCount?:  number;
}
```

---

## 9. Event Bus (Runtime ↔ UI)

Runtime emits events → `executionStore` reacts → UI re-renders:

| Event | Payload | UI Effect |
|---|---|---|
| `run:start` | `{ runId }` | Shows "Running" badge |
| `node:start` | `{ nodeId }` | Node pulses yellow |
| `node:success` | `{ nodeId, output }` | Node turns green |
| `node:error` | `{ nodeId, err }` | Node turns red + error badge |
| `node:skip` | `{ nodeId }` | Node dims gray |
| `node:retry` | `{ nodeId, attempt }` | Node shows retry count |
| `run:complete` | `{ outputs }` | All nodes settle |
| `run:abort` | `{ runId }` | All nodes reset |
| `log` | `{ nodeId, level, msg }` | Console tab entry |
| `breakpoint:hit` | `{ nodeId, input }` | Pause + highlight node |

---

## 10. WorkflowState (Persistent During Run)

```typescript
class WorkflowStateAccessor {
  private store: Map<string, unknown> = new Map();

  get(key: string): unknown     { return this.store.get(key); }
  set(key: string, val: unknown) { this.store.set(key, val); }
  delete(key: string)            { this.store.delete(key); }
  has(key: string): boolean      { return this.store.has(key); }
  clear()                        { this.store.clear(); }
}
// Created per runId — destroyed after run completes
```

---

## 11. Error Recovery Strategies

| Strategy | Config | Behavior |
|---|---|---|
| Fail Fast | `continueOnError: false` | Halts entire workflow on first error |
| Continue | `continueOnError: true` | Marks node error, continues downstream |
| Retry | `retryCount > 0` | Retries N times with delay before failing |
| Catch Branch | `try-catch` node | Routes error to catch-branch nodes |
| Fallback | Error port connection | Downstream node receives error object |
