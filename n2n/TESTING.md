# TESTING.md — N2N Toolz Testing Guide

---

## 1. Test Stack

| Layer | Tool |
|---|---|
| Unit Tests | Vitest |
| Component Tests | React Testing Library + Vitest |
| E2E Tests | Playwright |
| Node Definition Tests | Vitest + custom test harness |
| Code Gen Snapshot Tests | Vitest snapshots |
| Graph Engine Tests | Vitest |

---

## 2. Test Directory Structure

```
tests/
├── unit/
│   ├── engine/
│   │   ├── topoSort.test.ts
│   │   ├── graphStore.test.ts
│   │   ├── graphSerializer.test.ts
│   │   └── edgeValidator.test.ts
│   ├── codegen/
│   │   ├── scopeResolver.test.ts
│   │   ├── astBuilder.test.ts
│   │   ├── codeEmitter.test.ts
│   │   └── dependencyResolver.test.ts
│   ├── runtime/
│   │   ├── workflowRunner.test.ts
│   │   ├── nodeExecutor.test.ts
│   │   └── sandbox.test.ts
│   └── nodes/
│       ├── httpTrigger.test.ts
│       ├── ifElse.test.ts
│       ├── loopForEach.test.ts
│       └── ...
├── components/
│   ├── NodeRenderer.test.tsx
│   ├── NodeConfigForm.test.tsx
│   ├── PropertiesPanel.test.tsx
│   └── BottomPanel.test.tsx
├── codegen-snapshots/
│   ├── httpApi.snap
│   ├── cronJob.snap
│   └── complexFlow.snap
├── e2e/
│   ├── canvas.spec.ts
│   ├── workflow-run.spec.ts
│   └── export.spec.ts
└── fixtures/
    ├── workflows/
    │   ├── simple-api.n2nflow
    │   └── complex-flow.n2nflow
    └── plugins/
        └── test-plugin/
```

---

## 3. Running Tests

```bash
# All unit tests
npm test

# Watch mode
npm run test:watch

# With coverage
npm run test:coverage

# Component tests only
npm run test:components

# E2E tests (requires built app)
npm run test:e2e

# Specific file
npx vitest run tests/unit/engine/topoSort.test.ts
```

---

## 4. Unit Test: Graph Engine

```typescript
// tests/unit/engine/topoSort.test.ts
import { describe, it, expect } from 'vitest';
import { topologicalSort } from '../../../src/engine/graph/topoSort';
import { CyclicGraphError }  from '../../../src/engine/graph/errors';

describe('topologicalSort', () => {
  it('returns correct order for linear graph', () => {
    const nodes = [
      { id: 'A' }, { id: 'B' }, { id: 'C' },
    ];
    const edges = [
      { source: 'A', target: 'B' },
      { source: 'B', target: 'C' },
    ];
    const order = topologicalSort(nodes as any, edges as any);
    expect(order).toEqual(['A', 'B', 'C']);
  });

  it('throws CyclicGraphError for cyclic graph', () => {
    const nodes = [{ id: 'A' }, { id: 'B' }];
    const edges = [
      { source: 'A', target: 'B' },
      { source: 'B', target: 'A' },
    ];
    expect(() => topologicalSort(nodes as any, edges as any))
      .toThrowError(CyclicGraphError);
  });

  it('handles disconnected nodes', () => {
    const nodes = [{ id: 'A' }, { id: 'B' }, { id: 'C' }];
    const edges = [{ source: 'A', target: 'B' }];
    const order = topologicalSort(nodes as any, edges as any);
    expect(order).toContain('A');
    expect(order).toContain('B');
    expect(order).toContain('C');
    expect(order.indexOf('A')).toBeLessThan(order.indexOf('B'));
  });
});
```

---

## 5. Unit Test: Node Definition

```typescript
// tests/unit/nodes/httpRequest.test.ts
import { describe, it, expect, vi, beforeEach } from 'vitest';
import { HttpRequestNode } from '../../../src/nodes/core/HttpRequestNode';

const mockCtx = {
  nodeId:     'test-node',
  workflowId: 'test-wf',
  runId:      'test-run',
  env:        {},
  logger:     { info: vi.fn(), error: vi.fn(), warn: vi.fn(), debug: vi.fn() },
  abort:      new AbortController(),
  state:      { get: vi.fn(), set: vi.fn() },
};

describe('HttpRequestNode.execute()', () => {
  beforeEach(() => vi.clearAllMocks());

  it('returns response on 200', async () => {
    global.fetch = vi.fn().mockResolvedValue({
      ok:   true,
      status: 200,
      json: async () => ({ message: 'ok' }),
    });

    const result = await HttpRequestNode.execute(
      { body: { name: 'test' }, headers: {} },
      { url: 'https://api.test.com/data', method: 'POST', timeout: 5000, auth: 'none' },
      mockCtx
    );

    expect(result.response).toEqual({ message: 'ok' });
    expect(result.status).toBe(200);
  });

  it('throws on non-ok response', async () => {
    global.fetch = vi.fn().mockResolvedValue({ ok: false, status: 404 });
    await expect(
      HttpRequestNode.execute(
        { body: null, headers: {} },
        { url: 'https://api.test.com', method: 'GET', timeout: 5000, auth: 'none' },
        mockCtx
      )
    ).rejects.toThrow('404');
  });
});
```

---

## 6. Code Generation Snapshot Tests

```typescript
// tests/codegen-snapshots/httpApi.test.ts
import { describe, it, expect } from 'vitest';
import { CodeGenEngine }  from '../../../src/engine/codegen/CodeGenEngine';
import simpleApiWorkflow  from '../fixtures/workflows/simple-api.n2nflow';

describe('CodeGenEngine snapshots', () => {
  it('generates expected code for simple API workflow', async () => {
    const engine = new CodeGenEngine();
    const files  = await engine.generate(simpleApiWorkflow, {
      moduleSystem: 'esm',
      typescript:   false,
      prettier:     false,   // disable for stable snapshots
      comments:     false,
    });

    const indexFile = files.find(f => f.path === 'dist/index.js');
    expect(indexFile?.content).toMatchSnapshot();
  });
});
```

---

## 7. Workflow Runner Integration Tests

```typescript
// tests/unit/runtime/workflowRunner.test.ts
import { describe, it, expect } from 'vitest';
import { WorkflowRunner } from '../../../src/engine/runtime/WorkflowRunner';
import testWorkflow       from '../fixtures/workflows/simple-api.n2nflow';

describe('WorkflowRunner', () => {
  it('executes nodes in topological order', async () => {
    const executionOrder: string[] = [];
    const runner = new WorkflowRunner(testWorkflow, {
      onNodeStart: (nodeId) => executionOrder.push(nodeId),
    });

    await runner.run({ body: { name: 'Girish' } });

    // First node should be the trigger
    expect(executionOrder[0]).toBe('node_a1b2c3d4');
  });

  it('marks errored node and continues when continueOnError=true', async () => {
    // ...
  });

  it('aborts execution on stop()', async () => {
    // ...
  });
});
```

---

## 8. E2E Tests (Playwright)

```typescript
// tests/e2e/canvas.spec.ts
import { test, expect } from '@playwright/test';

test('user can add a node from palette to canvas', async ({ page }) => {
  await page.goto('http://localhost:5173');
  await page.click('[data-testid="new-project"]');
  await page.fill('[data-testid="project-name"]', 'Test Project');
  await page.click('[data-testid="create-project"]');

  // Drag HTTP Trigger from palette to canvas
  const palettItem = page.locator('[data-node-type="http-trigger"]');
  const canvas     = page.locator('[data-testid="canvas"]');
  await palettItem.dragTo(canvas);

  // Node should appear on canvas
  await expect(page.locator('[data-testid="node-http-trigger"]')).toBeVisible();
});

test('user can run a workflow and see success status', async ({ page }) => {
  // ...
  await page.keyboard.press('F5');
  await expect(page.locator('[data-node-status="success"]')).toHaveCount(3);
});
```

---

## 9. Coverage Targets

| Area | Target Coverage |
|---|---|
| Graph Engine | 95% |
| Code Generation | 90% |
| Node Definitions | 85% |
| Runtime | 85% |
| UI Components | 70% |
| IPC Handlers | 80% |

```bash
# View coverage report
npm run test:coverage
# Opens: coverage/index.html
```

---

## 10. Testing a Custom Plugin

```typescript
// Use the built-in test harness
import { createTestContext, runNode } from 'n2n-toolz/testing';

const ctx  = createTestContext({ env: { MY_API_KEY: 'test-key' } });

const result = await runNode(MyCustomNode, {
  input:  { query: 'hello' },
  config: { apiKey: 'test-key', endpoint: 'https://api.test.com', timeout: 5000 },
  ctx,
});

expect(result.data).toBeDefined();
```
