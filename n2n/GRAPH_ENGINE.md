# GRAPH_ENGINE.md — N2N Toolz Graph Engine

---

## 1. Core Data Structures

### 1.1 NodeModel

```typescript
interface NodeModel {
  id: string;                        // UUID v4 e.g. 'node_a1b2c3d4'
  type: string;                      // References NodeDefinition.type
  label: string;                     // User-editable display name
  position: { x: number; y: number };
  config: Record<string, unknown>;   // Current field values from Properties Panel
  enabled: boolean;                  // If false: node skipped during execution
  notes?: string;                    // Developer notes (markdown)
  executionOptions: {
    continueOnError: boolean;
    retryCount: number;              // 0–5
    retryDelay: number;              // ms
    timeout: number;                 // ms override (0 = use global default)
    alwaysExecute: boolean;          // Forces re-run even if input unchanged
  };
  uiState: {
    collapsed: boolean;
    width: number;
    breakpoint: boolean;
  };
}
```

### 1.2 EdgeModel

```typescript
interface EdgeModel {
  id: string;                    // UUID v4 e.g. 'edge_x9y8z7'
  source: string;                // Source NodeModel.id
  sourcePort: string;            // Output port name on source node
  target: string;                // Target NodeModel.id
  targetPort: string;            // Input port name on target node
  dataType?: DataType;           // Annotated type (inferred or user-set)
  label?: string;                // Optional visual edge label
  animated?: boolean;            // Animated during execution
}
```

### 1.3 WorkflowJSON (Serialized Graph)

```typescript
interface WorkflowJSON {
  id: string;
  name: string;
  description?: string;
  version: string;
  nodes: NodeModel[];
  edges: EdgeModel[];
  viewport: {
    x: number;
    y: number;
    zoom: number;
  };
  metadata: {
    created: string;              // ISO 8601
    modified: string;
    author: string;
    tags: string[];
  };
  variables: WorkflowVariable[];
  subWorkflows?: Record<string, WorkflowJSON>;
}

interface WorkflowVariable {
  name: string;
  type: DataType;
  defaultValue?: unknown;
  scope: 'workflow' | 'global';
}
```

---

## 2. Graph Store API

```typescript
interface GraphStore {
  // State
  nodes: Map<string, NodeModel>;
  edges: Map<string, EdgeModel>;
  selectedNodeIds: Set<string>;
  selectedEdgeIds: Set<string>;

  // Node mutations
  addNode(node: NodeModel): void;
  removeNode(id: string): void;                    // also removes connected edges
  updateNode(id: string, patch: Partial<NodeModel>): void;
  updateNodeConfig(id: string, config: Partial<Record<string, unknown>>): void;
  moveNode(id: string, position: {x: number; y: number}): void;
  duplicateNode(id: string): NodeModel;

  // Edge mutations
  addEdge(edge: EdgeModel): EdgeModel | ValidationError;
  removeEdge(id: string): void;

  // Selection
  selectNode(id: string, multi?: boolean): void;
  selectEdge(id: string, multi?: boolean): void;
  deselectAll(): void;
  selectAll(): void;

  // Graph queries
  getNode(id: string): NodeModel | undefined;
  getEdge(id: string): EdgeModel | undefined;
  getIncomingEdges(nodeId: string): EdgeModel[];
  getOutgoingEdges(nodeId: string): EdgeModel[];
  getAdjacentNodes(nodeId: string): NodeModel[];
  getRootNodes(): NodeModel[];           // Nodes with no inputs
  getTerminalNodes(): NodeModel[];       // Nodes with no outputs

  // Serialization
  serialize(): WorkflowJSON;
  deserialize(json: WorkflowJSON): void;
  toJSON(): string;

  // Validation
  validate(): GraphValidationResult;
}
```

---

## 3. Topological Sort

### Algorithm: Kahn's (BFS)

```typescript
function topologicalSort(nodes: NodeModel[], edges: EdgeModel[]): string[] {
  const inDegree = new Map<string, number>();
  const adjList = new Map<string, string[]>();

  // Initialize
  for (const node of nodes) {
    inDegree.set(node.id, 0);
    adjList.set(node.id, []);
  }

  // Build graph
  for (const edge of edges) {
    adjList.get(edge.source)!.push(edge.target);
    inDegree.set(edge.target, (inDegree.get(edge.target) ?? 0) + 1);
  }

  // BFS queue: all nodes with in-degree 0
  const queue: string[] = [];
  for (const [id, degree] of inDegree) {
    if (degree === 0) queue.push(id);
  }

  const order: string[] = [];
  while (queue.length > 0) {
    const current = queue.shift()!;
    order.push(current);
    for (const neighbor of adjList.get(current) ?? []) {
      const newDegree = inDegree.get(neighbor)! - 1;
      inDegree.set(neighbor, newDegree);
      if (newDegree === 0) queue.push(neighbor);
    }
  }

  // Cycle detection
  if (order.length !== nodes.length) {
    const cycleNodes = nodes.filter(n => !order.includes(n.id)).map(n => n.id);
    throw new CyclicGraphError(cycleNodes);
  }

  return order;
}
```

### Complexity

| Property | Value |
|---|---|
| Time | O(V + E) |
| Space | O(V + E) |
| Max graph size | 10,000 nodes before pagination recommended |
| Worker thread | Runs in Web Worker for graphs with > 500 nodes |

---

## 4. Edge Validation Rules

| Rule | Behavior |
|---|---|
| Source port must exist on source node type | Blocked — shows red edge |
| Target port must exist on target node type | Blocked — shows red edge |
| DataType mismatch (non-`any`) | Warning — yellow edge, allowed |
| Loop-back to ancestor node | Blocked — `CyclicGraphError` |
| Error port → non-error-handler node | Warning — yellow edge |
| Multiple edges to same non-`multiple` input port | Blocked |
| Multiple edges from same output port | Allowed (fan-out) |

---

## 5. Graph Serializer

```typescript
class GraphSerializer {
  serialize(store: GraphStore): WorkflowJSON {
    return {
      id: store.workflowId,
      name: store.workflowName,
      nodes: Array.from(store.nodes.values()),
      edges: Array.from(store.edges.values()),
      viewport: store.viewport,
      metadata: {
        modified: new Date().toISOString(),
        // ...
      },
      // ...
    };
  }

  deserialize(json: WorkflowJSON, store: GraphStore): void {
    store.nodes = new Map(json.nodes.map(n => [n.id, n]));
    store.edges = new Map(json.edges.map(e => [e.id, e]));
    store.viewport = json.viewport;
  }
}
```

---

## 6. Graph Validation

```typescript
interface GraphValidationResult {
  valid: boolean;
  errors: GraphError[];
  warnings: GraphWarning[];
}

interface GraphError {
  type: 'MISSING_CONFIG' | 'INVALID_CONNECTION' | 'CYCLIC_GRAPH' | 'UNKNOWN_NODE_TYPE';
  nodeId?: string;
  edgeId?: string;
  message: string;
}

// Validation checks performed:
// 1. All node types exist in NodeRegistry
// 2. All required config fields are filled
// 3. Graph is acyclic
// 4. All edges connect valid ports
// 5. At least one Trigger or Root node exists
// 6. All terminal nodes reached from root
```

---

## 7. Undo / Redo System

```typescript
// Snapshot-based undo (graphStore state)
interface HistoryStore {
  past:    WorkflowJSON[];   // Max 100 entries
  future:  WorkflowJSON[];
  current: WorkflowJSON;

  push(snapshot: WorkflowJSON): void;
  undo(): void;    // pops past → pushes to future → deserializes
  redo(): void;    // pops future → pushes to past → deserializes
  clear(): void;
}

// Auto-snapshot triggers:
// - addNode, removeNode, addEdge, removeEdge, updateNodeConfig, moveNode(debounced 500ms)
```

---

## 8. Sub-Workflow Composition

A **Sub-Workflow Node** wraps an entire child `WorkflowJSON` inside a parent graph node.

```typescript
interface SubWorkflowNode extends NodeModel {
  type: 'sub-workflow';
  config: {
    workflowId: string;           // references subWorkflows[id] in parent WorkflowJSON
    inputMapping:  Record<string, string>;   // parent input port → child variable
    outputMapping: Record<string, string>;  // child output → parent output port
  };
}
```

**Generated code for sub-workflow:**

```javascript
// Parent workflow calls child as async function
const node_sw_out = await runSubWorkflow_helpers(node_prev_out);
```

```javascript
// helpers.js — child workflow exported as function
export async function runSubWorkflow_helpers(input) {
  // ... child nodes ...
  return output;
}
```