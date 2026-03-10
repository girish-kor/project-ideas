# FOLDER_STRUCTURE.md — N2N Toolz Complete Folder Layout

---

## 1. Application Source Tree

```
n2n-toolz/
│
├── electron/                          ← Electron main process
│   ├── main.ts                        → App entry, BrowserWindow setup
│   ├── preload.ts                     → contextBridge IPC exposure
│   ├── ipc/                           → IPC handlers (Main Process side)
│   │   ├── project.ipc.ts
│   │   ├── codegen.ipc.ts
│   │   ├── runtime.ipc.ts
│   │   ├── plugin.ipc.ts
│   │   ├── deploy.ipc.ts
│   │   └── fs.ipc.ts
│   └── services/                      → Main-process-only services
│       ├── FileService.ts
│       ├── DevServerService.ts
│       └── DeployService.ts
│
├── src/                               ← Renderer process (React app)
│   ├── main.tsx                       → React entry point
│   ├── App.tsx
│   ├── routes.tsx
│   │
│   ├── components/                    ← UI components
│   │   ├── canvas/
│   │   │   ├── CanvasArea.tsx
│   │   │   ├── CanvasToolbar.tsx
│   │   │   ├── NodeRenderer.tsx
│   │   │   ├── NodeHeader.tsx
│   │   │   ├── PortHandle.tsx
│   │   │   ├── EdgeRenderer.tsx
│   │   │   ├── SelectionBox.tsx
│   │   │   └── Minimap.tsx
│   │   ├── panels/
│   │   │   ├── NodePalette.tsx
│   │   │   ├── NodePaletteItem.tsx
│   │   │   ├── PropertiesPanel.tsx
│   │   │   ├── NodeProperties.tsx
│   │   │   ├── EdgeProperties.tsx
│   │   │   ├── NodeConfigForm.tsx
│   │   │   ├── BottomPanel.tsx
│   │   │   ├── ConsoleTab.tsx
│   │   │   ├── ExecutionLogTab.tsx
│   │   │   ├── VariablesTab.tsx
│   │   │   ├── ErrorsTab.tsx
│   │   │   └── NetworkTab.tsx
│   │   ├── menus/
│   │   │   ├── TopMenuBar.tsx
│   │   │   ├── FileMenu.tsx
│   │   │   ├── EditMenu.tsx
│   │   │   ├── ViewMenu.tsx
│   │   │   ├── RunMenu.tsx
│   │   │   ├── DebugMenu.tsx
│   │   │   ├── DeployMenu.tsx
│   │   │   ├── PluginsMenu.tsx
│   │   │   ├── HelpMenu.tsx
│   │   │   ├── CanvasContextMenu.tsx
│   │   │   ├── NodeContextMenu.tsx
│   │   │   └── EdgeContextMenu.tsx
│   │   ├── modals/
│   │   │   ├── NewProjectModal.tsx
│   │   │   ├── ProjectSettingsModal.tsx
│   │   │   ├── ExportModal.tsx
│   │   │   ├── DeployModal.tsx
│   │   │   ├── NodeCodePreviewModal.tsx
│   │   │   ├── PluginInstallModal.tsx
│   │   │   ├── ShortcutsModal.tsx
│   │   │   └── PreferencesModal.tsx
│   │   ├── debug/
│   │   │   ├── BreakpointIndicator.tsx
│   │   │   ├── ExecutionTimeline.tsx
│   │   │   ├── VariableTree.tsx
│   │   │   └── WatchPanel.tsx
│   │   └── shared/
│   │       ├── Badge.tsx
│   │       ├── Tooltip.tsx
│   │       ├── Icon.tsx
│   │       ├── Spinner.tsx
│   │       ├── Toast.tsx
│   │       ├── Divider.tsx
│   │       ├── Kbd.tsx
│   │       ├── CodeBlock.tsx
│   │       ├── MonacoEditor.tsx
│   │       ├── SecretInput.tsx
│   │       ├── Toggle.tsx
│   │       ├── Select.tsx
│   │       ├── KeyValueEditor.tsx
│   │       └── FilePathPicker.tsx
│   │
│   ├── engine/                        ← Core logic (renderer-safe)
│   │   ├── graph/
│   │   │   ├── GraphStore.ts
│   │   │   ├── GraphSerializer.ts
│   │   │   ├── GraphValidator.ts
│   │   │   ├── EdgeValidator.ts
│   │   │   ├── topoSort.ts
│   │   │   └── errors.ts
│   │   ├── codegen/
│   │   │   ├── CodeGenEngine.ts
│   │   │   ├── GraphAnalyzer.ts
│   │   │   ├── DependencyResolver.ts
│   │   │   ├── ScopeResolver.ts
│   │   │   ├── ASTBuilder.ts
│   │   │   ├── ImportCollector.ts
│   │   │   ├── CodeEmitter.ts
│   │   │   ├── EntryPointBuilder.ts
│   │   │   ├── PackageJsonBuilder.ts
│   │   │   ├── EnvFileBuilder.ts
│   │   │   ├── IncrementalExporter.ts
│   │   │   └── templates/
│   │   │       ├── httpTrigger.ts
│   │   │       ├── cronTrigger.ts
│   │   │       ├── ifElse.ts
│   │   │       ├── loopForEach.ts
│   │   │       ├── loopWhile.ts
│   │   │       ├── tryCatch.ts
│   │   │       ├── merge.ts
│   │   │       ├── split.ts
│   │   │       ├── httpRequest.ts
│   │   │       ├── functionNode.ts
│   │   │       └── [all other node templates].ts
│   │   ├── runtime/
│   │   │   ├── WorkflowRunner.ts
│   │   │   ├── NodeExecutor.ts
│   │   │   ├── Sandbox.ts
│   │   │   ├── DebugInterceptor.ts
│   │   │   ├── RunReplayer.ts
│   │   │   └── EventBus.ts
│   │   └── plugins/
│   │       ├── NodeRegistry.ts
│   │       ├── PluginLoader.ts
│   │       ├── PluginValidator.ts
│   │       └── PluginSandbox.ts
│   │
│   ├── nodes/                         ← Built-in node definitions
│   │   ├── index.ts                   → registers all core nodes
│   │   ├── core/
│   │   │   ├── triggers/
│   │   │   │   ├── HttpTriggerNode.ts
│   │   │   │   ├── CronTriggerNode.ts
│   │   │   │   ├── ManualTriggerNode.ts
│   │   │   │   ├── EventTriggerNode.ts
│   │   │   │   └── WebSocketTriggerNode.ts
│   │   │   ├── flow/
│   │   │   │   ├── IfElseNode.ts
│   │   │   │   ├── SwitchNode.ts
│   │   │   │   ├── LoopForEachNode.ts
│   │   │   │   ├── LoopWhileNode.ts
│   │   │   │   ├── TryCatchNode.ts
│   │   │   │   ├── MergeNode.ts
│   │   │   │   ├── SplitNode.ts
│   │   │   │   └── DelayNode.ts
│   │   │   ├── transform/
│   │   │   │   ├── MapNode.ts
│   │   │   │   ├── FilterNode.ts
│   │   │   │   ├── ReduceNode.ts
│   │   │   │   ├── JsonParseNode.ts
│   │   │   │   ├── JsonStringifyNode.ts
│   │   │   │   ├── SetVariableNode.ts
│   │   │   │   └── TemplateNode.ts
│   │   │   ├── output/
│   │   │   │   ├── HttpResponseNode.ts
│   │   │   │   ├── ConsoleLogNode.ts
│   │   │   │   ├── ReturnValueNode.ts
│   │   │   │   └── ThrowErrorNode.ts
│   │   │   ├── http/
│   │   │   │   ├── HttpRequestNode.ts
│   │   │   │   ├── GraphQLClientNode.ts
│   │   │   │   └── OAuth2HandlerNode.ts
│   │   │   ├── database/
│   │   │   │   ├── MySQLQueryNode.ts
│   │   │   │   ├── PostgresQueryNode.ts
│   │   │   │   ├── MongoDBQueryNode.ts
│   │   │   │   ├── SQLiteQueryNode.ts
│   │   │   │   └── RedisNode.ts
│   │   │   ├── fs/
│   │   │   │   ├── ReadFileNode.ts
│   │   │   │   ├── WriteFileNode.ts
│   │   │   │   ├── DeleteFileNode.ts
│   │   │   │   ├── ListDirectoryNode.ts
│   │   │   │   └── WatchFileNode.ts
│   │   │   ├── auth/
│   │   │   │   ├── JWTSignNode.ts
│   │   │   │   ├── JWTVerifyNode.ts
│   │   │   │   ├── HashBcryptNode.ts
│   │   │   │   └── AESCipherNode.ts
│   │   │   ├── messaging/
│   │   │   │   ├── SMTPEmailNode.ts
│   │   │   │   ├── SlackMessageNode.ts
│   │   │   │   ├── TelegramNode.ts
│   │   │   │   └── KafkaNode.ts
│   │   │   └── code/
│   │   │       ├── FunctionNode.ts
│   │   │       └── ShellCommandNode.ts
│   │   └── icons/                     → SVG icons for each node
│   │
│   ├── stores/                        ← Zustand state stores
│   │   ├── graphStore.ts
│   │   ├── uiStore.ts
│   │   ├── executionStore.ts
│   │   ├── projectStore.ts
│   │   ├── pluginStore.ts
│   │   ├── historyStore.ts
│   │   ├── debugStore.ts
│   │   └── settingsStore.ts
│   │
│   ├── types/                         ← TypeScript interfaces
│   │   ├── graph.types.ts
│   │   ├── node.types.ts
│   │   ├── codegen.types.ts
│   │   ├── runtime.types.ts
│   │   ├── plugin.types.ts
│   │   ├── ui.types.ts
│   │   └── ipc.types.ts
│   │
│   └── utils/
│       ├── idGenerator.ts
│       ├── portValidator.ts
│       ├── hashJSON.ts
│       ├── envResolver.ts
│       ├── evalCondition.ts
│       └── sleep.ts
│
├── tests/                             ← All test files (see TESTING.md)
├── plugins/                           ← Installed plugin packages
├── projects/                          ← User project files
├── dist/                              ← Electron build output
├── releases/                          ← Packaged installers
│
├── package.json
├── vite.config.ts
├── tsconfig.json
├── tsconfig.electron.json
├── electron-builder.config.ts
├── vitest.config.ts
├── .eslintrc.ts
├── .prettierrc
└── .gitignore
```

---

## 2. Generated Project Workspace

```
~/n2n-toolz-workspace/
├── projects/
│   └── my-project.n2n/               ← See FILE_FORMAT.md
├── plugins/
│   ├── n2n-plugin-stripe/
│   └── n2n-plugin-twilio/
├── exports/
│   └── my-project/
│       └── 2026-03-10T12-00-00/
│           ├── dist/
│           └── project-export.zip
└── logs/
    ├── app-2026-03-10.log
    └── runtime-2026-03-10.log
```

