# CODEGEN.md — N2N Toolz Code Generation Engine

---

## 1. Pipeline Overview

```
WorkflowJSON
     │
     ▼
┌─────────────────┐
│ GraphAnalyzer   │  → validates graph, resolves node types, marks disabled nodes
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ DependencyRslvr │  → maps node types → npm packages → builds package.json
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ ScopeResolver   │  → assigns variable names to each node output in order
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ ASTBuilder      │  → calls node.generateCode() per node, wraps in function/route
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ ImportCollector │  → deduplicates and sorts import statements
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ CodeEmitter     │  → recast.print(AST) → JS string
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Formatter       │  → prettier.format(code, { parser: 'babel' | 'typescript' })
└────────┬────────┘
         │
         ▼
  Generated Files
  dist/index.js
  dist/routes/*.js
  dist/workflows/*.js
  dist/lib/helpers.js
  package.json
  .env.example
```

---

## 2. Scope Resolution

### Variable Naming Convention

| Element | Pattern | Example |
|---|---|---|
| Node output | `n_{shortId}_out` | `n_a1b2_out` |
| Node error | `n_{shortId}_err` | `n_a1b2_err` |
| Loop iterator | `n_{shortId}_item` | `n_a1b2_item` |
| Loop index | `n_{shortId}_idx` | `n_a1b2_idx` |
| Loop accumulator | `n_{shortId}_acc` | `n_a1b2_acc` |
| Merged branches | `n_{shortId}_merged` | `n_a1b2_merged` |
| Condition result | `n_{shortId}_cond` | `n_a1b2_cond` |

`shortId` = first 8 chars of node UUID.

### ScopeMap (passed to each generateCode call)

```typescript
interface ScopeMap {
  inputVars: Record<string, string>;  // portName → resolved variable name from upstream
  outputVar: string;                  // variable name this node should assign its result to
  errorVar:  string;                  // variable name for this node's error
}

// Example for node_002 (JSON Parse) receiving from node_001 (HTTP Trigger):
{
  inputVars: { raw: 'n_node001_out.response' },
  outputVar: 'n_node002_out',
  errorVar:  'n_node002_err'
}
```

---

## 3. AST Builder

### 3.1 Core Builder Methods

```typescript
class ASTBuilder {
  // Variable declarations
  const(name: string, init: Expression): VariableDeclaration;
  let(name: string, init?: Expression): VariableDeclaration;

  // Expressions
  await(expr: Expression): AwaitExpression;
  call(callee: string, args: Expression[]): CallExpression;
  member(obj: string, prop: string): MemberExpression;
  templateLiteral(parts: string[], exprs: Expression[]): TemplateLiteral;
  arrow(params: string[], body: Expression | BlockStatement): ArrowFunctionExpression;

  // Statements
  ifElse(test: Expression, consequent: Statement[], alternate?: Statement[]): IfStatement;
  forOf(item: string, collection: Expression, body: Statement[]): ForOfStatement;
  tryCatch(tryBlock: Statement[], catchParam: string, catchBlock: Statement[]): TryStatement;
  return(expr: Expression): ReturnStatement;
  throw(expr: Expression): ThrowStatement;

  // Functions
  asyncFunction(name: string, params: string[], body: Statement[]): FunctionDeclaration;

  // Imports
  importDefault(name: string, from: string): ImportDeclaration;
  importNamed(names: string[], from: string): ImportDeclaration;
}
```

### 3.2 Generated File Skeleton

```typescript
// Template for HTTP Trigger workflow
const fileSkeleton = `
import express from 'express';
${collectedImports}

const router = express.Router();

router.${method}('${path}', async (req, res, next) => {
  try {
    ${nodeStatements}
  } catch (__workflowErr) {
    next(__workflowErr);
  }
});

export default router;
`;
```

---

## 4. Node Code Templates

### 4.1 HTTP Request Node

```javascript
// Template
const {outputVar} = await (async () => {
  const __res = await fetch({url}, {
    method: '{method}',
    headers: { 'Content-Type': 'application/json', ...{inputVars.headers} },
    body: {method} !== 'GET' ? JSON.stringify({inputVars.body}) : undefined,
    signal: AbortSignal.timeout({timeout}),
  });
  if (!__res.ok) throw new Error(`HTTP {method} {url} failed: ${__res.status}`);
  return { response: await __res.json(), status: __res.status };
})();
```

### 4.2 If / Else Node

```javascript
let {outputVar_true}, {outputVar_false};
if ({condition_with_resolved_vars}) {
  {outputVar_true} = {inputVars.data};
} else {
  {outputVar_false} = {inputVars.data};
}
```

### 4.3 Loop (forEach) Node

```javascript
const {outputVar} = [];
for (const [{itemVar}, {indexVar}] of ({inputVars.collection} ?? []).entries()) {
  // inner nodes
  {outputVar}.push({innerResult});
}
```

### 4.4 Try / Catch Node

```javascript
let {outputVar_success}, {outputVar_error};
try {
  // try-branch nodes
  {outputVar_success} = {tryBranchResult};
} catch ({errorVar}) {
  {outputVar_error} = {errorVar};
}
```

### 4.5 Merge Node (Promise.all)

```javascript
const {outputVar} = await Promise.{strategy}([
  {branch1Promise},
  {branch2Promise},
  // ...
]);
```

### 4.6 Function Node

```javascript
const {outputVar} = await (async (input) => {
  {userCode}
})({inputVars.data});
```

---

## 5. Import Collector

Tracks all required imports and deduplicates:

```typescript
class ImportCollector {
  private imports: Map<string, Set<string>> = new Map();
  // key: module name, value: Set of specifiers

  add(module: string, specifier?: string): void {
    if (!this.imports.has(module)) this.imports.set(module, new Set());
    if (specifier) this.imports.get(module)!.add(specifier);
  }

  emit(): string {
    return Array.from(this.imports.entries())
      .sort(([a], [b]) => a.localeCompare(b))
      .map(([mod, specs]) =>
        specs.size === 0
          ? `import '${mod}';`
          : specs.size === 1 && [...specs][0] === '__default'
          ? `import ${[...specs][0].replace('__default', mod.split('/').pop()!)} from '${mod}';`
          : `import { ${[...specs].join(', ')} } from '${mod}';`
      )
      .join('\n');
  }
}
```

---

## 6. Dependency Resolution

```typescript
// Complete map: node type → npm dependencies
const NODE_DEPS: Record<string, { packages: string[]; version: string }[]> = {
  'http-trigger':      [{ packages: ['express'],           version: '^4.18.0' }],
  'http-request':      [{ packages: [],                    version: '' }],  // uses native fetch
  'cron-trigger':      [{ packages: ['node-cron'],         version: '^3.0.0' }],
  'mysql-query':       [{ packages: ['mysql2'],            version: '^3.0.0' }],
  'postgres-query':    [{ packages: ['pg'],                version: '^8.0.0' }],
  'mongodb-query':     [{ packages: ['mongoose'],          version: '^8.0.0' }],
  'redis-get-set':     [{ packages: ['ioredis'],           version: '^5.0.0' }],
  'jwt-sign':          [{ packages: ['jsonwebtoken'],      version: '^9.0.0' }],
  'jwt-verify':        [{ packages: ['jsonwebtoken'],      version: '^9.0.0' }],
  'hash-bcrypt':       [{ packages: ['bcryptjs'],          version: '^2.4.3' }],
  'aes-cipher':        [{ packages: [],                    version: '' }],  // uses node:crypto
  'smtp-email':        [{ packages: ['nodemailer'],        version: '^6.0.0' }],
  'slack-message':     [{ packages: ['@slack/webhook'],    version: '^7.0.0' }],
  'kafka-producer':    [{ packages: ['kafkajs'],           version: '^2.0.0' }],
  'graphql-client':    [{ packages: ['graphql-request'],   version: '^6.0.0' }],
  'template':          [{ packages: ['handlebars'],        version: '^4.7.0' }],
  'shell-command':     [{ packages: [],                    version: '' }],  // uses node:child_process
};
```

---

## 7. Generated package.json

```typescript
function generatePackageJson(usedNodeTypes: string[], projectMeta: ProjectMeta): string {
  const deps: Record<string, string> = {};

  for (const type of usedNodeTypes) {
    const nodeDeps = NODE_DEPS[type] ?? [];
    for (const { packages, version } of nodeDeps) {
      for (const pkg of packages) {
        deps[pkg] = version;
      }
    }
  }

  return JSON.stringify({
    name: projectMeta.name.toLowerCase().replace(/\s+/g, '-'),
    version: '1.0.0',
    description: `Generated by N2N Toolz`,
    type: 'module',
    main: 'index.js',
    scripts: {
      start: 'node index.js',
      dev: 'node --watch index.js',
    },
    dependencies: deps,
    engines: { node: '>=18.0.0' },
  }, null, 2);
}
```

---

## 8. Incremental Code Generation (Cache)

```typescript
// Only regenerate nodes whose config or upstream connections changed
class CodeGenCache {
  private cache: Map<string, { hash: string; code: string }> = new Map();

  isCached(nodeId: string, node: NodeModel, incomingEdges: EdgeModel[]): boolean {
    const hash = this.computeHash(node, incomingEdges);
    return this.cache.get(nodeId)?.hash === hash;
  }

  getCached(nodeId: string): string {
    return this.cache.get(nodeId)!.code;
  }

  store(nodeId: string, node: NodeModel, incomingEdges: EdgeModel[], code: string): void {
    const hash = this.computeHash(node, incomingEdges);
    this.cache.set(nodeId, { hash, code });
  }

  private computeHash(node: NodeModel, edges: EdgeModel[]): string {
    return JSON.stringify({ config: node.config, edges: edges.map(e => e.id) });
  }
}
```

---

## 9. TypeScript Output Mode

When `Settings → Code Generation → Enable TypeScript` is on:

- Imports use `.ts` extensions
- Variable declarations get inferred types from port `DataType`
- Node outputs typed as `Record<string, unknown>` if type is `any`
- `tsconfig.json` emitted with `"module": "ESNext"`, `"target": "ES2022"`
- Types file `dist/types/n2n.d.ts` generated with all workflow input/output shapes
