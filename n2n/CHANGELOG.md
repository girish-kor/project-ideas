# CHANGELOG.md — N2N Toolz Version History

All notable changes follow [Semantic Versioning](https://semver.org/) and [Keep a Changelog](https://keepachangelog.com/) format.

---

## [Unreleased]

### Added
- TypeScript output mode (in progress)
- Monaco editor for Function Node

---

## [1.0.0] — 2026-03-10

### Added

**Core Platform**
- Electron desktop app (Windows, macOS, Linux)
- React Flow-based infinite canvas
- Node Palette with 50+ built-in nodes
- Properties Panel with dynamic config form rendering
- Bottom panel with Console, Execution Log, Variables, Errors, Network tabs
- Workflow tab bar (multi-workflow per project)
- Minimap widget

**Node Library**
- 5 Trigger nodes: HTTP, Cron, Manual, Event, WebSocket
- 8 Flow Control nodes: If/Else, Switch, Loop forEach, While Loop, Try/Catch, Merge, Split, Delay
- 7 Data Transform nodes: Map, Filter, Reduce, JSON Parse, JSON Stringify, Set Variable, Template
- 4 Output nodes: HTTP Response, Console Log, Return Value, Throw Error
- 3 HTTP/API nodes: HTTP Request, GraphQL Client, OAuth2 Handler
- 5 Database nodes: MySQL, PostgreSQL, MongoDB, SQLite, Redis
- 5 File System nodes: Read, Write, Delete, List Directory, Watch
- 4 Auth/Security nodes: JWT Sign, JWT Verify, Hash (bcrypt), AES Cipher
- 5 Messaging nodes: SMTP Email, Slack, Telegram, Twilio, Kafka
- 2 Custom Code nodes: Function Node, Shell Command

**Code Generation**
- Graph → Node.js ESM source code generation
- CommonJS (CJS) output mode
- Prettier formatting of all generated files
- Auto-inferred `package.json` with correct dependencies
- `.env.example` generation from secret config fields
- Secret field → `process.env.VAR_NAME` resolution
- Incremental export (only regenerates changed workflows)

**Runtime & Debugging**
- Embedded workflow runner (vm2 sandbox)
- Breakpoints with optional condition expressions
- Step over / step into debug controls
- Variable Explorer (runtime scope viewer)
- NodeLogger per node (5 log levels)
- Network Inspector for HTTP nodes
- Run Replay (re-execute with same input)
- Execution timeline with per-node duration

**Plugin System**
- ESM dynamic import plugin loader
- Plugin validation + security scanning
- Plugin trust levels (trusted / community / local)
- Plugin registry URL configurable
- `n2n-toolz/testing` test harness for plugin authors

**Project & File System**
- `.n2n` project format (folder-based)
- `.n2nflow` workflow JSON format
- `.n2nenv` environment variable files
- `plugins.json` plugin references
- `.n2nignore` export exclusion file
- Auto-save with configurable interval
- Snapshot-based undo/redo (100 history steps)

**Deployment**
- Local file copy deployment
- Dev Server with hot-reload (child process)
- SSH/SCP deployment with pre/post commands
- Docker export (Dockerfile + docker-compose.yml)
- AWS Lambda export (zip + handler wrapper)
- GCP Cloud Run deploy script generation
- Deployment history log

**Security**
- vm2 sandboxed execution
- Module whitelist for sandboxed nodes
- Secret field encryption (AES-256-GCM in project file)
- Parameterized SQL generation
- `helmet` auto-injected in HTTP apps
- IPC contextBridge (no nodeIntegration)

---

## [0.9.0-beta] — 2026-02-01

### Added
- Initial beta release (internal only)
- Canvas, basic node palette, code generation MVP
- HTTP Trigger, HTTP Request, If/Else, JSON Parse, HTTP Response nodes
- Basic project file format
- Simple embedded runtime (no sandbox)

### Known Issues
- Cyclic graph not detected (fixed in 1.0.0)
- Plugin system not yet implemented
- No undo/redo

---

## [0.5.0-alpha] — 2025-12-15

### Added
- Proof of concept: visual canvas → Node.js code generation
- 10 core nodes
- No persistence, no plugins, no runtime

---

*Versions older than 0.5.0 are internal prototypes and not documented.*
