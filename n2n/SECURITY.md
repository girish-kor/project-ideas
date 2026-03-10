# SECURITY.md — N2N Toolz Security Model

---

## 1. Threat Model

| Threat | Mitigation |
|---|---|
| Malicious plugin code | Sandbox isolation + module whitelist |
| Secret leakage in codegen | `secret` fields → env var references only |
| Arbitrary file access from nodes | fs access restricted in sandbox mode |
| Prototype pollution via JSON | `JSON.parse` inputs sanitized |
| XSS via node labels/config | React escapes all rendered strings |
| Code injection in Function Node | vm2 sandbox with `eval: false` |
| Credential logging | NodeLogger redacts `secret` field values |

---

## 2. Sandbox Security (Embedded Runtime)

```typescript
const vm = new VM({
  timeout:  30000,
  eval:     false,        // no eval()
  wasm:     false,        // no WebAssembly
  fixAsync: true,
  sandbox: {
    require: createSafeRequire([
      'path', 'url', 'crypto', 'util',
      'querystring', 'buffer', 'events',
      'stream', 'assert',
      // NO: fs, child_process, net, http, os
    ]),
    fetch:   globalThis.fetch,
    console: nodeLogger,
    process: {
      env:      safeEnv,           // filtered env — only whitelisted keys
      version:  process.version,
      platform: process.platform,
      // NO: process.exit, process.kill, process.binding
    },
  },
});
```

### Module Whitelist (Sandbox Mode)

| Module | Allowed | Reason |
|---|---|---|
| `path` | ✅ | Path manipulation (safe) |
| `url` | ✅ | URL parsing (safe) |
| `crypto` | ✅ | Hashing, HMAC (safe) |
| `util` | ✅ | Utilities (safe) |
| `buffer` | ✅ | Binary data |
| `stream` | ✅ | Data streaming |
| `fs` | ❌ | File system — blocked in sandbox |
| `child_process` | ❌ | Shell execution — blocked |
| `net` | ❌ | Raw sockets — blocked |
| `os` | ❌ | System info — blocked |

---

## 3. Secret Field Handling

### In Properties Panel
- `secret: true` fields rendered as `<input type="password">`
- Values stored encrypted in project file (AES-256-GCM, key derived from machine ID)

### In Code Generation
```typescript
// secret fields NEVER inlined:
// ❌ BAD: const apiKey = 'sk-abc123';
// ✅ GOOD: const apiKey = process.env.MY_NODE_API_KEY;

function resolveConfigValue(field: FieldSchema, value: unknown): string {
  if (field.secret) {
    const envKey = deriveEnvKey(field);
    return `process.env.${envKey}`;
  }
  return JSON.stringify(value);
}
```

### In Logs
```typescript
function sanitizeForLog(config: NodeConfig, schema: Record<string, FieldSchema>): NodeConfig {
  return Object.fromEntries(
    Object.entries(config).map(([k, v]) => [
      k,
      schema[k]?.secret ? '[REDACTED]' : v,
    ])
  );
}
```

---

## 4. Env Variable Security

| File | Committed to VCS? | Encrypted? |
|---|---|---|
| `.env.example` | ✅ Yes (no real values) | No |
| `.env.dev` | ⚠️ User's choice | Optional |
| `.env.prod` | ❌ Never (in .gitignore) | Optional |
| `project.json` secret values | ❌ No (encrypted in file) | Yes (AES-256) |

---

## 5. Plugin Security

### Validation Before Registration

```typescript
function validatePlugin(plugin: PluginDefinition): void {
  // 1. Schema check
  if (plugin.n2n?.type !== 'node-plugin') throw new PluginValidationError('...');

  // 2. No dangerous globals used in generateCode templates
  for (const node of plugin.nodes) {
    const codeStr = node.generateCode.toString();
    const banned = ['process.exit', 'require(', '__dirname', 'eval(', 'Function('];
    for (const b of banned) {
      if (codeStr.includes(b)) throw new PluginSecurityError(`Banned pattern: ${b}`);
    }
  }

  // 3. Sandbox execute functions (vm2 wrapping for untrusted plugins)
  if (!plugin.trusted) {
    wrapExecuteInSandbox(plugin);
  }
}
```

### Plugin Trust Levels

| Level | Source | execute() isolation |
|---|---|---|
| Trusted | Official N2N registry | Native (no extra sandbox) |
| Community | npm / GitHub | vm2 wrapped |
| Local Dev | Local path | vm2 wrapped + warnings shown |

---

## 6. IPC Security

- All IPC handlers validate payload schema before processing
- `ipcRenderer` exposed only via `contextBridge` in `preload.ts`
- No `nodeIntegration: true` in renderer — uses context isolation

```typescript
// preload.ts
contextBridge.exposeInMainWorld('n2nAPI', {
  project: {
    open:  (path: string)   => ipcRenderer.invoke('project:open', { path }),
    save:  (data: unknown)  => ipcRenderer.invoke('project:save', data),
  },
  // ... only whitelisted IPC channels exposed
});
```

---

## 7. Generated Code Security

### SQL Injection Prevention

Generated MySQL/Postgres node code always uses parameterized queries:

```javascript
// ✅ Always generated (parameterized)
const n_001_out = await db.query('SELECT * FROM users WHERE id = ?', [userId]);

// ❌ Never generated (string interpolation)
// const n_001_out = await db.query(`SELECT * FROM users WHERE id = ${userId}`);
```

### HTTP Response Headers

Generated Express apps include security headers via auto-injected `helmet`:

```javascript
// Auto-added to index.js when HTTP Trigger nodes are present
import helmet from 'helmet';
app.use(helmet());
```

---

## 8. Reporting Security Issues

Email: `security@n2ntoolz.io`
PGP Key: Available at `https://n2ntoolz.io/.well-known/pgp-key.txt`

Do not report security issues via public GitHub issues.