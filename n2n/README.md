# N2N Toolz 🔗

> **Node-to-Node Visual Development Platform for Node.js**
> Build entire Node.js applications through visual workflows — no boilerplate, no setup friction.

---

## What Is N2N Toolz?

N2N Toolz is a desktop-based visual IDE where developers drag, drop, and connect **nodes** to build Node.js applications. Every visual connection generates clean, production-ready Node.js source code.

```
[Drag Nodes] → [Connect Ports] → [Configure] → [Generate Node.js Code] → [Deploy]
```

---

## Key Features

| Feature | Description |
|---|---|
| 🎨 Visual Canvas | React Flow-powered infinite canvas for building workflows |
| ⚡ Live Execution | Run workflows directly inside the app — no export needed |
| 🧱 Node Library | 50+ built-in core nodes covering HTTP, DB, Auth, Files, Messaging |
| 🔌 Plugin System | Build and install custom nodes as npm packages |
| 📦 Code Export | Generates clean ESM / CJS / TypeScript Node.js source |
| 🐛 Visual Debugger | Breakpoints, variable watch, execution timeline |
| 🚀 One-Click Deploy | Deploy to local, SSH, Docker, AWS Lambda, GCP |
| 🗂️ Multi-Workflow | Multiple workflow tabs per project |

---

## Quick Start

### Prerequisites

```bash
Node.js >= 18.x
npm >= 9.x
```

### Install

```bash
git clone https://github.com/your-org/n2n-toolz.git
cd n2n-toolz
npm install
npm run dev
```

### Build Desktop App

```bash
npm run build:electron
```

---

## Core Workflow (5 Steps)

1. **Open N2N Toolz** → Create new project
2. **Drag nodes** from Node Palette onto canvas
3. **Connect ports** — output → input via drag
4. **Configure nodes** via Properties Panel (right sidebar)
5. **Run or Export** — live run or generate Node.js code

---

## Documentation Index

| File | Description |
|---|---|
| [BLUEPRINT.md](./BLUEPRINT.md) | Full system architecture |
| [MENUBAR.md](./MENUBAR.md) | UI menu structure |
| [INSTALLATION.md](./INSTALLATION.md) | Setup and install guide |
| [ARCHITECTURE.md](./ARCHITECTURE.md) | Deep-dive architecture layers |
| [NODES.md](./NODES.md) | All built-in node reference |
| [NODE_SCHEMA.md](./NODE_SCHEMA.md) | Node definition schema spec |
| [GRAPH_ENGINE.md](./GRAPH_ENGINE.md) | Graph data structure and engine |
| [CODEGEN.md](./CODEGEN.md) | Code generation system |
| [RUNTIME.md](./RUNTIME.md) | Workflow execution runtime |
| [PLUGIN_GUIDE.md](./PLUGIN_GUIDE.md) | Custom plugin development |
| [API.md](./API.md) | IPC and REST API reference |
| [STATE_MANAGEMENT.md](./STATE_MANAGEMENT.md) | App state architecture |
| [UI_COMPONENTS.md](./UI_COMPONENTS.md) | UI component reference |
| [FILE_FORMAT.md](./FILE_FORMAT.md) | .n2n project file format |
| [EXPORT.md](./EXPORT.md) | Export targets and pipeline |
| [DEPLOYMENT.md](./DEPLOYMENT.md) | Deployment guide |
| [DEBUGGING.md](./DEBUGGING.md) | Debugging system |
| [SECURITY.md](./SECURITY.md) | Security model |
| [TESTING.md](./TESTING.md) | Testing guide |
| [ENVIRONMENT.md](./ENVIRONMENT.md) | Env variables reference |
| [FOLDER_STRUCTURE.md](./FOLDER_STRUCTURE.md) | Project folder layout |
| [SHORTCUTS.md](./SHORTCUTS.md) | Keyboard shortcuts |
| [GLOSSARY.md](./GLOSSARY.md) | Terms and definitions |
| [CONTRIBUTING.md](./CONTRIBUTING.md) | Contribution guide |
| [ROADMAP.md](./ROADMAP.md) | Planned features |
| [CHANGELOG.md](./CHANGELOG.md) | Version history |

---

## Tech Stack

```
Electron + Node.js     → Desktop shell
React 18 + TypeScript  → UI layer
React Flow             → Canvas engine
Zustand + Immer        → State management
recast / escodegen     → AST + code generation
vm2                    → Sandboxed execution
Vite + esbuild         → Build tooling
Prettier               → Code formatting
```

---

## License

MIT © N2N Toolz Contributors
