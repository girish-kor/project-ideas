# SHORTCUTS.md — N2N Toolz Keyboard Shortcuts

---

## 1. Global Shortcuts

| Action | Windows / Linux | macOS |
|---|---|---|
| New Project | `Ctrl+N` | `Cmd+N` |
| Open Project | `Ctrl+O` | `Cmd+O` |
| Save | `Ctrl+S` | `Cmd+S` |
| Save As | `Ctrl+Shift+S` | `Cmd+Shift+S` |
| Export Code | `Ctrl+E` | `Cmd+E` |
| Preferences | `Ctrl+,` | `Cmd+,` |
| Toggle Full Screen | `F11` | `Cmd+Ctrl+F` |
| Close Tab | `Ctrl+W` | `Cmd+W` |
| Quit App | `Ctrl+Q` | `Cmd+Q` |

---

## 2. Canvas Shortcuts

| Action | Windows / Linux | macOS |
|---|---|---|
| Undo | `Ctrl+Z` | `Cmd+Z` |
| Redo | `Ctrl+Y` / `Ctrl+Shift+Z` | `Cmd+Shift+Z` |
| Cut | `Ctrl+X` | `Cmd+X` |
| Copy | `Ctrl+C` | `Cmd+C` |
| Paste | `Ctrl+V` | `Cmd+V` |
| Duplicate | `Ctrl+D` | `Cmd+D` |
| Delete | `Delete` / `Backspace` | `Delete` / `Backspace` |
| Select All | `Ctrl+A` | `Cmd+A` |
| Deselect | `Escape` | `Escape` |
| Find Node | `Ctrl+F` | `Cmd+F` |
| Zoom In | `Ctrl++` / Scroll Up | `Cmd++` / Scroll Up |
| Zoom Out | `Ctrl+-` / Scroll Down | `Cmd+-` / Scroll Down |
| Fit to Screen | `Ctrl+0` | `Cmd+0` |
| Reset Zoom | `Ctrl+1` | `Cmd+1` |
| Toggle Grid | `Ctrl+G` | `Cmd+G` |
| Toggle Snap | `Ctrl+Shift+G` | `Cmd+Shift+G` |
| Group Selection | `Ctrl+G` (with nodes selected) | `Cmd+G` |
| Add Note | `N` | `N` |
| Multi-select (hold) | `Shift+Click` | `Shift+Click` |
| Pan Canvas (hold) | `Space+Drag` | `Space+Drag` |
| Box Select (drag) | `Drag on empty canvas` | `Drag on empty canvas` |

---

## 3. Node Shortcuts

| Action | Shortcut |
|---|---|
| Rename selected node | `F2` |
| Toggle enable/disable | `E` |
| Toggle breakpoint | `B` |
| Run from this node | `Ctrl+Shift+R` |
| Inspect output | `I` |
| Open in code view | `Ctrl+Shift+C` |
| Collapse / Expand | `C` |

---

## 4. Execution Shortcuts

| Action | Windows / Linux | macOS |
|---|---|---|
| Run Workflow | `F5` | `F5` |
| Stop Execution | `F6` / `Ctrl+F5` | `F6` / `Cmd+F5` |
| Pause Execution | `F7` | `F7` |
| Step Over | `F10` | `F10` |
| Step Into | `F11` | `F11` |
| Continue (from breakpoint) | `F5` | `F5` |
| Validate Workflow | `Ctrl+Shift+V` | `Cmd+Shift+V` |
| Clear Execution State | `Ctrl+Shift+X` | `Cmd+Shift+X` |
| Replay Last Run | `Ctrl+Shift+R` | `Cmd+Shift+R` |

---

## 5. Panel & View Shortcuts

| Action | Windows / Linux | macOS |
|---|---|---|
| Toggle Node Palette | `Ctrl+B` | `Cmd+B` |
| Toggle Properties Panel | `Ctrl+P` | `Cmd+P` |
| Toggle Bottom Panel | `` Ctrl+` `` | `` Cmd+` `` |
| Toggle Console Tab | `Ctrl+J` | `Cmd+J` |
| Focus Search (Palette) | `Ctrl+Shift+F` | `Cmd+Shift+F` |
| Switch to Next Tab | `Ctrl+Tab` | `Ctrl+Tab` |
| Switch to Prev Tab | `Ctrl+Shift+Tab` | `Ctrl+Shift+Tab` |
| Toggle Minimap | `Ctrl+M` | `Cmd+M` |

---

## 6. Debug Shortcuts

| Action | Shortcut |
|---|---|
| Add/Remove Breakpoint on selected node | `B` |
| Remove All Breakpoints | `Ctrl+Shift+B` |
| Add Watch Expression | `Ctrl+Shift+W` |
| Open Execution Timeline | `Ctrl+Shift+T` |
| Open Error Log | `Ctrl+Shift+E` |

---

## 7. Customizing Shortcuts

All shortcuts are customizable:

```
Settings (Ctrl+,) → Keyboard Shortcuts → [click action] → [press new keys]
```

**Constraints:**
- Cannot override OS-level shortcuts
- Cannot assign duplicate shortcuts within the same context
- Reset to defaults: Settings → Keyboard Shortcuts → Reset All

---

## 8. Quick Node Search (Command Palette)

`Ctrl+Shift+P` opens the command palette with fuzzy search over:

- All node types (drag to canvas on enter)
- All menu actions
- Recent workflows
- Open project files

```
> http req
──────────────────────────────────
  📦 HTTP Request Node    (add to canvas)
  📦 HTTP Trigger Node    (add to canvas)
  ▶  Run Workflow
  📂 Open: my-api.n2nflow
```
