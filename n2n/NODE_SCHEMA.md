# NODE_SCHEMA.md — Node Definition Schema Specification

> Every node in N2N Toolz — core or plugin — **must** implement this schema.

---

## 1. Full NodeDefinition Interface

```typescript
interface NodeDefinition {
  // ── Identity ───────────────────────────────────────────────────
  type: string;               // Unique kebab-case key. e.g. 'http-request'
  version: string;            // semver e.g. '1.0.0'
  category: NodeCategory;     // See Category Enum below
  label: string;              // Human-readable name
  description: string;        // One-line description shown in palette tooltip
  icon: string;               // Relative SVG path or inline SVG string
  color?: string;             // Hex color for node header. Default: per category
  tags?: string[];            // Search tags e.g. ['api', 'rest', 'http']
  deprecated?: boolean;       // Hides from palette, still renderable

  // ── Ports ──────────────────────────────────────────────────────
  inputs: PortDefinition[];
  outputs: PortDefinition[];

  // ── Config Schema ──────────────────────────────────────────────
  configSchema: Record<string, FieldSchema>;
  defaultConfig?: Record<string, unknown>;

  // ── Runtime Execution ──────────────────────────────────────────
  execute: NodeExecuteFn;

  // ── Code Generation ────────────────────────────────────────────
  generateCode: NodeCodeGenFn;

  // ── Lifecycle Hooks (optional) ─────────────────────────────────
  onInit?:     (config: NodeConfig) => void | Promise<void>;
  onDestroy?:  (config: NodeConfig) => void | Promise<void>;
  validate?:   (config: NodeConfig) => ValidationResult;

  // ── UI Hints (optional) ────────────────────────────────────────
  resizable?: boolean;
  minWidth?:  number;
  maxWidth?:  number;
  testable?:  boolean;        // Shows "Test Node" button in properties panel
  docs?:      string;         // URL to external documentation
}
```

---

## 2. Port Definition

```typescript
interface PortDefinition {
  name: string;               // Port identifier key (used in edge connections)
  label?: string;             // Display label (defaults to name)
  type: DataType;             // Type constraint for connection validation
  required?: boolean;         // Default: false
  multiple?: boolean;         // Allow multiple incoming connections to this port
  description?: string;       // Shown as tooltip on hover
  hidden?: boolean;           // Port exists but not shown visually
}

type DataType =
  | 'any'
  | 'string'
  | 'number'
  | 'boolean'
  | 'object'
  | 'array'
  | 'stream'
  | 'buffer'
  | 'error'
  | 'null';
```

---

## 3. Field Schema (Config Form Renderer)

```typescript
interface FieldSchema {
  type: FieldType;
  label: string;
  description?: string;       // Helper text shown below field
  default?: unknown;
  required?: boolean;
  placeholder?: string;
  secret?: boolean;           // Masks input + excludes from exported code (uses env var)
  disabled?: boolean;
  options?: FieldOption[];    // For 'select' and 'multiselect' types
  min?: number;               // For 'number' type
  max?: number;               // For 'number' type
  step?: number;              // For 'number' type (slider)
  rows?: number;              // For 'textarea' and 'code' type height
  language?: string;          // For 'code' type: 'javascript' | 'json' | 'sql'
  validation?: {
    pattern?: string;         // Regex string
    message?: string;         // Error message on validation fail
    custom?: string;          // JS expression referencing `value`
  };
  dependsOn?: {               // Conditionally show field
    field: string;
    value: unknown;
  };
}

type FieldType =
  | 'string'          // Single-line text input
  | 'number'          // Numeric input
  | 'boolean'         // Toggle switch
  | 'select'          // Dropdown single select
  | 'multiselect'     // Dropdown multi select (returns array)
  | 'textarea'        // Multi-line text
  | 'code'            // Monaco code editor (inline)
  | 'json'            // JSON editor with validation
  | 'secret'          // Password-masked input → resolves to env var in codegen
  | 'expression'      // JS expression editor (small, single-line)
  | 'file-path'       // File picker
  | 'color'           // Color picker
  | 'keyvalue';       // Key-value pair editor (produces object)

interface FieldOption {
  label: string;
  value: string | number | boolean;
  description?: string;
}
```

---

## 4. Node Execute Function

```typescript
type NodeExecuteFn = (
  input: NodeInput,
  config: NodeConfig,
  ctx: ExecutionContext
) => Promise<NodeOutput>;

interface NodeInput {
  [portName: string]: unknown;
}

interface NodeOutput {
  [portName: string]: unknown;
}

interface NodeConfig {
  [fieldName: string]: unknown;
}

interface ExecutionContext {
  nodeId:     string;
  workflowId: string;
  runId:      string;
  env:        Record<string, string>;
  logger:     NodeLogger;
  state:      WorkflowStateAccessor;
  abort:      AbortController;
}
```

---

## 5. Node Code Generation Function

```typescript
type NodeCodeGenFn = (
  node: NodeModel,
  scope: ScopeMap,
  ctx: CodeGenContext
) => CodeGenResult;

interface ScopeMap {
  inputVars:  Record<string, string>;   // portName → variable name in generated scope
  outputVar:  string;                   // variable name for this node's output
  errorVar:   string;                   // variable name for this node's error
}

interface CodeGenContext {
  workflowId: string;
  emitImport:  (pkg: string, specifier: string) => void;  // adds import to file header
  emitHelper:  (name: string, code: string) => void;      // adds helper function to file
}

type CodeGenResult =
  | string          // Raw JS code string (will be parsed into AST)
  | ASTNode         // Pre-built AST node (from recast builders)
  | CodeGenResult[] // Multiple statements
```

---

## 6. Node Category Enum

```typescript
type NodeCategory =
  | 'Trigger'
  | 'Flow Control'
  | 'Data Transform'
  | 'Output'
  | 'HTTP / API'
  | 'Database'
  | 'File System'
  | 'Auth & Security'
  | 'Messaging'
  | 'Custom Code'
  | 'Integrations'    // Plugin-installed nodes
  | 'Utilities';
```

---

## 7. Validation Result

```typescript
interface ValidationResult {
  valid: boolean;
  errors: ValidationError[];
}

interface ValidationError {
  field: string;    // config field key with error
  message: string;
  severity: 'error' | 'warning';
}
```

---

## 8. Minimal Node Example

```typescript
// src/nodes/core/HelloWorldNode.ts

export const HelloWorldNode: NodeDefinition = {
  type: 'hello-world',
  version: '1.0.0',
  category: 'Utilities',
  label: 'Hello World',
  description: 'Outputs a greeting message',
  icon: './icons/hello.svg',

  inputs: [
    { name: 'name', type: 'string', required: false }
  ],
  outputs: [
    { name: 'message', type: 'string' },
    { name: 'error',   type: 'error'  }
  ],

  configSchema: {
    greeting: {
      type: 'string',
      label: 'Greeting Word',
      default: 'Hello',
      required: true,
    },
  },

  execute: async (input, config) => {
    const name = input.name ?? 'World';
    return { message: `${config.greeting}, ${name}!` };
  },

  generateCode: (node, scope, ctx) => {
    ctx.emitImport('// no imports needed', '');
    return `
      const ${scope.outputVar} = {
        message: \`${node.config.greeting}, \${${scope.inputVars.name} ?? 'World'}!\`
      };
    `;
  },
};
```

---

## 9. Node Registration

```typescript
// src/nodes/index.ts
import { NodeRegistry } from '../engine/plugins/NodeRegistry';
import { HelloWorldNode } from './core/HelloWorldNode';
// ... all other core nodes

NodeRegistry.getInstance().registerAll([
  HelloWorldNode,
  // ...
]);
```

---

## 10. Schema Validation Rules (enforced by PluginLoader)

| Rule | Detail |
|---|---|
| `type` must be unique | Duplicate type throws `NodeConflictError` |
| `type` format | Must match `/^[a-z][a-z0-9-]*$/` |
| At least one output port | Every node must have ≥ 1 output OR be terminal (type ends in `-response`, `-log`, `-return`, `-error`) |
| `execute` required | Must be async function |
| `generateCode` required | Must return string or ASTNode |
| `secret` fields in config | Auto-referenced as `process.env.FIELD_NAME` in generated code — never inlined |
