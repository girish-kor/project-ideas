# ROADMAP.md — N2N Toolz Development Roadmap

---

## Version 1.0.0 — Foundation ✅

| Feature | Status |
|---|---|
| Visual canvas (React Flow) | ✅ Complete |
| Core node library (50+ nodes) | ✅ Complete |
| Code generation (ESM/CJS) | ✅ Complete |
| Embedded runtime (vm2) | ✅ Complete |
| Plugin system | ✅ Complete |
| Breakpoint debugger | ✅ Complete |
| Project file format (.n2n / .n2nflow) | ✅ Complete |
| SSH/Docker deployment | ✅ Complete |
| Electron desktop app | ✅ Complete |

---

## Version 1.1.0 — Developer Experience 🔧

**Target: Q2 2026**

| Feature | Priority | Description |
|---|---|---|
| TypeScript output mode | High | Generate `.ts` files with full typing |
| Monaco editor in Function Node | High | Full syntax highlighting + autocomplete |
| Node test button | High | Test individual node with manual input |
| Import from n8n JSON | Medium | Convert n8n workflows to N2N Toolz |
| Canvas search / jump-to | Medium | `Ctrl+F` to find and highlight nodes |
| Node grouping / containers | Medium | Visual grouping for related nodes |
| Workflow comments | Low | Add sticky notes to canvas |
| Multi-cursor node editing | Low | Edit same field on multiple nodes at once |

---

## Version 1.2.0 — Collaboration & Cloud 🌐

**Target: Q3 2026**

| Feature | Priority | Description |
|---|---|---|
| Cloud project sync | High | Save/load projects from cloud storage |
| Real-time collaboration | High | Multiple users on same canvas simultaneously |
| Version history (Git-based) | High | Per-workflow version history with diffs |
| Workflow sharing (public URLs) | Medium | Shareable read-only workflow links |
| Team workspaces | Medium | Organization-level project management |
| Plugin marketplace UI | Medium | In-app plugin registry browser |
| Workflow templates library | Low | Pre-built workflow templates |

---

## Version 1.3.0 — Advanced Nodes & Integrations 🔌

**Target: Q4 2026**

| Feature | Priority | Description |
|---|---|---|
| gRPC Client node | High | Call gRPC services visually |
| WebSocket server node | High | Full WebSocket server setup |
| GraphQL server node | Medium | Auto-generate GraphQL schema from nodes |
| Message Queue nodes | Medium | RabbitMQ, SQS, Azure Service Bus |
| S3 / Cloud Storage nodes | Medium | AWS S3, GCS, Azure Blob operations |
| OpenAI / LLM nodes | Medium | GPT, Claude, Gemini API nodes |
| Stripe / Payment nodes | Low | Stripe webhooks + payment operations |
| Google Sheets node | Low | Read/write Google Sheets |
| Airtable node | Low | Airtable operations |

---

## Version 2.0.0 — Web Platform 🚀

**Target: 2027**

| Feature | Description |
|---|---|
| Browser-based app | Full web app (no Electron required) |
| Cloud execution | Run workflows directly in cloud without exporting |
| Workflow marketplace | Community-shared workflows |
| API Gateway integration | One-click deploy as managed API |
| Monitoring dashboard | Live metrics for deployed workflows |
| Schedule management UI | View/manage all cron jobs centrally |
| RBAC / Access control | Role-based access for team features |
| White-label SDK | Embed N2N Toolz canvas in other apps |

---

## Backlog (No Version Assigned)

| Feature | Description |
|---|---|
| Visual diff for workflows | Side-by-side workflow comparison |
| AI-assisted node suggestion | Suggest next node based on context |
| Performance profiling | Flame graph for workflow execution |
| Generated code linting | Run ESLint on generated code at export |
| Workflow unit testing UI | Visual test case builder |
| Offline plugin cache | Use plugins without internet |
| Node execution mocking | Mock node outputs for testing downstream |

---

## Deprecation Schedule

| Item | Deprecated In | Removed In | Migration |
|---|---|---|---|
| `vm2` sandbox | 1.2.0 | 2.0.0 | Replaced by `isolated-vm` |
| `.n2nflow` v0 format | 1.1.0 | 1.3.0 | Auto-migrated on open |
| CJS output mode (default) | 1.1.0 | 2.0.0 | ESM becomes only default |

---

## Contributing to Roadmap

Open a GitHub Discussion tagged `roadmap` to propose new features.
Vote on features by 👍 reacting to existing issues.
