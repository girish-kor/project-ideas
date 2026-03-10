# MENUBAR.md — N2N Toolz UI Menu Structure

> **N2N Toolz** | Visual Node-to-Node Development Platform for Node.js
> Version: 1.0.0 | Last Updated: 2026-03-10

---

## 1. Top Navigation Bar

| Menu Label | Purpose | Key Actions |
|---|---|---|
| **File** | Project lifecycle management | New, Open, Save, Export, Close |
| **Edit** | Canvas & node editing operations | Undo, Redo, Cut, Copy, Paste, Select All |
| **View** | UI layout and display controls | Zoom, Minimap, Grid, Panel toggles |
| **Run** | Workflow execution controls | Run, Stop, Step, Validate |
| **Debug** | Debugging tools and breakpoints | Add Breakpoint, Watch, Inspect |
| **Deploy** | Deployment pipeline triggers | Build, Package, Deploy, Publish |
| **Plugins** | Manage external/custom node packs | Install, Manage, Create Plugin |
| **Help** | Documentation and support | Docs, Shortcuts, About, Report Issue |

---

### 1.1 File Menu

```
File
├── New Project               → Opens project wizard dialog
├── New Workflow              → Creates blank canvas in current project
├── Open Project...           → File picker → loads .n2n project bundle
├── Open Recent ▶             → Flyout: last 10 opened projects
├── Save                      → Ctrl+S → saves current state to disk
├── Save As...                → Fork current project to new path
├── Export ▶
│   ├── Export as Node.js     → Generates /dist/*.js source files
│   ├── Export as ZIP         → Bundles project + generated code
│   ├── Export as Docker      → Generates Dockerfile + source
│   └── Export as JSON Graph  → Raw workflow graph JSON
├── Import ▶
│   ├── Import Workflow JSON  → Load external .n2nflow file
│   └── Import from n8n       → Parse & convert n8n JSON
├── Project Settings...       → Opens project config modal
└── Exit                      → Confirm dialog → close app
```

---

### 1.2 Edit Menu

```
Edit
├── Undo                      → Ctrl+Z → reverts last canvas action
├── Redo                      → Ctrl+Y → reapplies reverted action
├── Cut                       → Removes selected nodes to clipboard
├── Copy                      → Copies selected node(s) to clipboard
├── Paste                     → Pastes from clipboard at cursor
├── Duplicate                 → Clones selected node(s) in place
├── Delete                    → Removes selected nodes + connected edges
├── Select All                → Ctrl+A → selects all canvas nodes
├── Deselect All              → Escape → clears selection
├── Find Node...              → Ctrl+F → search by node name/type
└── Preferences...            → Opens app-level settings modal
```

---

### 1.3 View Menu

```
View
├── Zoom In                   → Ctrl++ → scales canvas up
├── Zoom Out                  → Ctrl+- → scales canvas down
├── Fit to Screen             → Ctrl+0 → fits all nodes in viewport
├── Reset Zoom                → Returns to 100% zoom
├── Toggle Grid               → Show/hide canvas grid overlay
├── Toggle Snap to Grid       → Enable/disable node snapping
├── Toggle Minimap            → Show/hide bottom-right minimap widget
├── Toggle Node Palette       → Show/hide left-side node library panel
├── Toggle Properties Panel   → Show/hide right-side node config panel
├── Toggle Console            → Show/hide bottom debug console
├── Toggle Workflow List      → Show/hide workflow tab sidebar
├── Theme ▶
│   ├── Dark Mode
│   ├── Light Mode
│   └── System Default
└── Full Screen               → F11 toggle
```

---

### 1.4 Run Menu

```
Run
├── Run Workflow              → F5 → executes current workflow
├── Run from Selected Node    → Executes from highlighted node forward
├── Stop Execution            → F6 → halts active runtime
├── Pause Execution           → Breakpoint-aware pause
├── Step Over                 → Advances one node step
├── Step Into                 → Enters sub-workflow node
├── Validate Workflow         → Checks schema, connections, missing configs
└── Clear Execution State     → Resets all node execution highlights
```

---

### 1.5 Debug Menu

```
Debug
├── Add Breakpoint            → Toggle breakpoint on selected node
├── Remove All Breakpoints    → Clears all canvas breakpoints
├── Watch Expression...       → Adds variable to watch panel
├── Inspect Node Output       → Shows last output payload of selected node
├── Execution Timeline        → Opens timing waterfall view
├── Variable Explorer         → Opens runtime variable scope panel
├── Error Log                 → Opens filterable error log window
└── Replay Last Run           → Re-executes last workflow with same input
```

---

### 1.6 Deploy Menu

```
Deploy
├── Build Project             → Compiles workflow → Node.js dist/
├── Run Build Script          → Executes custom pre/post build hooks
├── Start Dev Server          → Spawns local Node.js dev server
├── Stop Dev Server           → Kills active dev server process
├── Deploy ▶
│   ├── Deploy to Local       → Copies build to configured local path
│   ├── Deploy to SSH Target  → Pushes via SSH/SCP to remote
│   ├── Deploy to Docker      → Builds & runs Docker container
│   └── Deploy to Cloud ▶
│       ├── AWS Lambda
│       ├── GCP Cloud Run
│       └── Custom Endpoint
├── Deployment History        → Shows log of past deployments
└── Configure Targets...      → Opens deployment targets config modal
```

---

## 2. Left Sidebar — Node Library Panel

### 2.1 Panel Header

| Control | Action |
|---|---|
| Search bar | Real-time fuzzy search across all node types |
| Filter icon | Filter by category, tags, or source (core/plugin) |
| Sort toggle | Sort alphabetically or by most-used |
| Collapse all | Collapses all category groups |

---

### 2.2 Node Library Categories

```
Node Library
├── 📦 Core
│   ├── Trigger Nodes
│   │   ├── HTTP Trigger
│   │   ├── Cron Trigger
│   │   ├── Event Trigger
│   │   ├── Manual Trigger
│   │   └── WebSocket Trigger
│   ├── Flow Control
│   │   ├── If / Else
│   │   ├── Switch
│   │   ├── Loop (forEach)
│   │   ├── While Loop
│   │   ├── Try / Catch
│   │   ├── Merge
│   │   ├── Split
│   │   └── Delay
│   ├── Data Transform
│   │   ├── Map
│   │   ├── Filter
│   │   ├── Reduce
│   │   ├── JSON Parse
│   │   ├── JSON Stringify
│   │   ├── Set Variable
│   │   └── Template (Handlebars)
│   └── Output
│       ├── Response (HTTP)
│       ├── Console Log
│       ├── Return Value
│       └── Throw Error
│
├── 🌐 HTTP / API
│   ├── HTTP Request (GET/POST/PUT/DELETE)
│   ├── REST Client
│   ├── GraphQL Client
│   ├── Webhook Sender
│   ├── OAuth2 Handler
│   └── API Rate Limiter
│
├── 🗄️ Database
│   ├── MySQL Query
│   ├── PostgreSQL Query
│   ├── MongoDB Query
│   ├── SQLite Query
│   ├── Redis Get/Set
│   └── ORM Model Node
│
├── 📁 File System
│   ├── Read File
│   ├── Write File
│   ├── Delete File
│   ├── List Directory
│   ├── Watch File/Dir
│   └── Compress/Extract (ZIP)
│
├── 🔐 Auth & Security
│   ├── JWT Sign
│   ├── JWT Verify
│   ├── Hash (bcrypt/sha256)
│   ├── Encrypt/Decrypt (AES)
│   └── Session Manager
│
├── 📨 Messaging
│   ├── SMTP Email
│   ├── Slack Message
│   ├── Telegram Message
│   ├── Twilio SMS
│   └── Kafka Producer/Consumer
│
├── ⚙️ Custom Code
│   ├── Function Node (inline JS)
│   ├── Shell Command
│   ├── Child Process Spawn
│   └── Eval Expression
│
├── 🔌 Integrations (Plugin Nodes)
│   └── [Installed plugins appear here]
│
└── ⭐ Favorites
    └── [User-pinned nodes]
```

---

## 3. Right Sidebar — Properties Panel

```
Properties Panel (context-sensitive)
├── [No Selection]            → Shows canvas properties (zoom, grid settings)
├── [Node Selected]
│   ├── Node Info
│   │   ├── Node Name (editable label)
│   │   ├── Node Type (read-only)
│   │   └── Node ID (read-only UUID)
│   ├── Configuration
│   │   └── [Dynamic fields per node type — rendered from node schema]
│   ├── Input Ports
│   │   └── Port list with type labels
│   ├── Output Ports
│   │   └── Port list with type labels
│   ├── Execution Options
│   │   ├── Enable/Disable node
│   │   ├── Always Execute toggle
│   │   └── Retry on Failure (count + delay)
│   └── Notes
│       └── Markdown text area for developer notes
└── [Edge Selected]
    ├── Edge ID (read-only)
    ├── Source Port
    ├── Target Port
    └── Data Type annotation
```

---

## 4. Bottom Panel — Console & Debug Tabs

```
Bottom Panel
├── Console Tab
│   ├── Output stream (stdout/stderr per node execution)
│   ├── Clear button
│   └── Filter by node / log level
├── Execution Log Tab
│   ├── Timeline list of executed nodes
│   ├── Status badges (✅ success / ❌ error / ⏳ pending)
│   └── Duration per node
├── Variables Tab
│   ├── Runtime variable scope tree
│   └── Editable values (dev mode)
├── Errors Tab
│   ├── Error list with stack traces
│   └── Jump-to-node button per error
└── Network Tab
    ├── HTTP request/response log
    └── Request inspector (headers, body, status)
```

---

## 5. Workflow Tab Bar (Top of Canvas)

```
Workflow Tabs
├── [Tab per open workflow]   → Click to switch, × to close
├── + New Workflow Tab        → Creates blank workflow
└── Overflow menu (▼)        → Shows hidden tabs when overflow
```

---

## 6. Settings Menu (Accessible via Gear Icon or Edit > Preferences)

```
Settings
├── General
│   ├── Default project directory
│   ├── Auto-save interval
│   └── Language / Locale
├── Editor
│   ├── Grid size
│   ├── Snap sensitivity
│   ├── Default zoom level
│   └── Node default size
├── Execution
│   ├── Default timeout per node (ms)
│   ├── Max retry count
│   └── Sandbox mode toggle
├── Code Generation
│   ├── Output directory path
│   ├── Code style (ESM / CommonJS)
│   ├── Enable TypeScript output
│   └── Prettier formatting toggle
├── Plugins
│   ├── Plugin registry URL
│   ├── Installed plugins list
│   ├── Auto-update plugins toggle
│   └── Plugin sandbox toggle
├── Deployment
│   ├── Saved deployment targets
│   └── SSH key management
├── Appearance
│   ├── Theme selector
│   ├── Font size
│   └── Animation speed
└── Keyboard Shortcuts
    └── Editable keybinding table
```

---

## 7. Context Menu (Right-Click on Canvas / Node)

### 7.1 Canvas Right-Click
```
├── Paste
├── Select All
├── Add Note
├── Fit to Screen
└── Canvas Properties
```

### 7.2 Node Right-Click
```
├── Run This Node
├── Add Breakpoint / Remove Breakpoint
├── Inspect Output
├── Duplicate
├── Copy
├── Delete
├── Disable / Enable
├── Group Selection
├── Rename
└── View Generated Code → opens code preview modal for this node
```

### 7.3 Edge Right-Click
```
├── Delete Connection
├── Inspect Data Type
└── Add Middleware Node (inserts transform between two nodes)
```

---

*End of MENUBAR.md*