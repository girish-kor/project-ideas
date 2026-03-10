# PLUGIN_GUIDE.md — N2N Toolz Custom Plugin Development

---

## 1. Plugin Overview

A plugin is an **npm package** that exports one or more `NodeDefinition` objects. Once installed, its nodes appear in the Node Palette under "Integrations" or a custom category.

---

## 2. Plugin Package Structure

```
my-n2n-plugin/
├── package.json
├── index.js              ← Plugin entry point
├── nodes/
│   ├── MyNode.js         ← Node definition
│   └── AnotherNode.js
├── icons/
│   ├── my-node.svg
│   └── another-node.svg
├── test/
│   └── MyNode.test.js
└── README.md
```

---

## 3. package.json Requirements

```json
{
  "name": "n2n-plugin-my-integration",
  "version": "1.0.0",
  "description": "My custom N2N Toolz plugin",
  "type": "module",
  "main": "index.js",
  "n2n": {
    "type": "node-plugin",
    "minAppVersion": "1.0.0",
    "category": "My Integration",
    "icon": "./icons/plugin-icon.svg"
  },
  "keywords": ["n2n-plugin", "n2n-toolz"],
  "peerDependencies": {
    "n2n-toolz": ">=1.0.0"
  }
}
```

> `"n2n": { "type": "node-plugin" }` is **required** — PluginLoader rejects without it.

---

## 4. Plugin Entry Point (index.js)

```javascript
// index.js
import { MyNode }      from './nodes/MyNode.js';
import { AnotherNode } from './nodes/AnotherNode.js';

export default {
  name:    'my-integration',
  version: '1.0.0',
  author:  'Your Name',
  nodes:   [MyNode, AnotherNode],

  // Optional lifecycle hooks
  onInstall:   async () => { /* setup */ },
  onUninstall: async () => { /* cleanup */ },
};
```

---

## 5. Node Definition (Full Example)

```javascript
// nodes/MyNode.js

export const MyNode = {
  // ── Identity ──────────────────────────────────────────────
  type:        'my-integration-fetch',
  version:     '1.0.0',
  category:    'My Integration',
  label:       'My API Fetch',
  description: 'Fetches data from My Integration API',
  icon:        new URL('../icons/my-node.svg', import.meta.url).pathname,
  tags:        ['api', 'fetch', 'my-integration'],

  // ── Ports ─────────────────────────────────────────────────
  inputs: [
    { name: 'query',   type: 'string',  required: true },
    { name: 'options', type: 'object',  required: false },
  ],
  outputs: [
    { name: 'data',  type: 'object' },
    { name: 'error', type: 'error'  },
  ],

  // ── Config Schema ─────────────────────────────────────────
  configSchema: {
    apiKey: {
      type:     'secret',
      label:    'API Key',
      required: true,
      secret:   true,       // → process.env.MY_INTEGRATION_API_KEY in generated code
    },
    baseUrl: {
      type:        'string',
      label:       'Base URL',
      default:     'https://api.my-integration.com/v1',
      placeholder: 'https://api.example.com',
    },
    timeout: {
      type:    'number',
      label:   'Timeout (ms)',
      default: 10000,
      min:     1000,
      max:     60000,
    },
    method: {
      type:    'select',
      label:   'HTTP Method',
      default: 'GET',
      options: [
        { label: 'GET',  value: 'GET'  },
        { label: 'POST', value: 'POST' },
      ],
    },
  },

  // ── Live Execution ────────────────────────────────────────
  execute: async (input, config, ctx) => {
    const { query, options = {} } = input;
    const url = `${config.baseUrl}/search?q=${encodeURIComponent(query)}`;

    try {
      const response = await fetch(url, {
        method:  config.method,
        headers: { Authorization: `Bearer ${config.apiKey}` },
        signal:  AbortSignal.timeout(config.timeout),
        ...options,
      });

      if (!response.ok) {
        throw new Error(`API error: ${response.status} ${response.statusText}`);
      }

      const data = await response.json();
      ctx.logger.info(`Fetched ${data.results?.length ?? 0} results`);
      return { data };
    } catch (err) {
      ctx.logger.error('Fetch failed', err);
      return { error: err };
    }
  },

  // ── Code Generation ───────────────────────────────────────
  generateCode: (node, scope, ctx) => {
    ctx.emitImport('// native fetch — no import needed', '');

    return `
      let ${scope.outputVar}, ${scope.errorVar};
      try {
        const __url = \`\${process.env.MY_INTEGRATION_BASE_URL ?? '${node.config.baseUrl}'}/search?q=\${encodeURIComponent(${scope.inputVars.query})}\`;
        const __res = await fetch(__url, {
          method: '${node.config.method}',
          headers: { Authorization: \`Bearer \${process.env.MY_INTEGRATION_API_KEY}\` },
          signal: AbortSignal.timeout(${node.config.timeout}),
        });
        if (!__res.ok) throw new Error(\`API error: \${__res.status}\`);
        ${scope.outputVar} = { data: await __res.json() };
      } catch (__err) {
        ${scope.errorVar} = __err;
        ${scope.outputVar} = { error: __err };
      }
    `;
  },

  // ── Validation ────────────────────────────────────────────
  validate: (config) => {
    const errors = [];
    if (!config.apiKey) {
      errors.push({ field: 'apiKey', message: 'API Key is required', severity: 'error' });
    }
    if (!config.baseUrl?.startsWith('https://')) {
      errors.push({ field: 'baseUrl', message: 'Base URL must use HTTPS', severity: 'warning' });
    }
    return { valid: errors.length === 0, errors };
  },

  testable: true,
  docs: 'https://docs.my-integration.com/n2n-plugin',
};
```

---

## 6. Installing Plugins

### From npm Registry

```bash
# Via N2N Toolz UI: Plugins → Install Plugin → paste package name
# Or via terminal:
npm install n2n-plugin-my-integration --prefix ~/.n2n-toolz/plugins
```

### From Local Path (Development)

```bash
# In Settings → Plugins → Load from path
# Or symlink for live development:
npm link /path/to/my-n2n-plugin --prefix ~/.n2n-toolz/plugins
```

### From GitHub

```bash
npm install github:username/my-n2n-plugin --prefix ~/.n2n-toolz/plugins
```

---

## 7. Plugin Loading Internals

```typescript
class PluginLoader {
  async loadAll(pluginsDir: string): Promise<void> {
    const dirs = await fs.readdir(pluginsDir);
    await Promise.allSettled(dirs.map(d => this.loadOne(path.join(pluginsDir, d))));
  }

  async loadOne(pluginPath: string): Promise<void> {
    const pkg = JSON.parse(await fs.readFile(path.join(pluginPath, 'package.json'), 'utf8'));

    // Validation
    if (pkg.n2n?.type !== 'node-plugin') throw new PluginValidationError('Not a node plugin');
    if (this.isVersionCompatible(pkg.n2n.minAppVersion) === false)
      throw new PluginVersionError(pkg.name, pkg.n2n.minAppVersion);

    // Load entry point
    const plugin = (await import(path.join(pluginPath, pkg.main))).default;
    this.validate(plugin);

    // Register nodes
    for (const nodeDef of plugin.nodes) {
      NodeRegistry.getInstance().register(nodeDef);
    }

    // Call lifecycle hook
    await plugin.onInstall?.();
    this.loaded.set(pkg.name, plugin);
  }
}
```

---

## 8. Plugin Development Workflow

```bash
# 1. Scaffold a new plugin
npx create-n2n-plugin my-plugin-name
cd n2n-plugin-my-plugin-name

# 2. Run tests
npm test

# 3. Link to N2N Toolz for live testing
npm link
cd ~/.n2n-toolz/plugins
npm link n2n-plugin-my-plugin-name

# 4. Reload plugins in app: Plugins → Reload All Plugins

# 5. Publish
npm publish
```

---

## 9. Plugin Constraints

| Constraint | Limit |
|---|---|
| Max nodes per plugin | 100 |
| Node type conflicts | Not allowed — throws `NodeConflictError` |
| `secret` config fields | Auto-redacted from logs and generated code |
| File system access in `execute()` | Allowed only outside sandbox mode |
| Network access in `execute()` | Allowed (use `ctx.abort` for cancellation) |
| `eval()` in `generateCode()` | Not allowed — use string templates |

---

## 10. Testing Plugins

```javascript
// test/MyNode.test.js
import { describe, it, expect, vi } from 'vitest';
import { MyNode } from '../nodes/MyNode.js';

describe('MyNode execute()', () => {
  it('returns data on success', async () => {
    global.fetch = vi.fn().mockResolvedValue({
      ok: true,
      json: async () => ({ results: [{ id: 1 }] }),
    });

    const result = await MyNode.execute(
      { query: 'test' },
      { apiKey: 'fake-key', baseUrl: 'https://api.example.com', method: 'GET', timeout: 5000 },
      { logger: { info: vi.fn(), error: vi.fn() }, abort: new AbortController() }
    );

    expect(result.data.results).toHaveLength(1);
  });

  it('returns error on HTTP failure', async () => {
    global.fetch = vi.fn().mockResolvedValue({ ok: false, status: 404, statusText: 'Not Found' });
    const result = await MyNode.execute(
      { query: 'test' },
      { apiKey: 'fake-key', baseUrl: 'https://api.example.com', method: 'GET', timeout: 5000 },
      { logger: { info: vi.fn(), error: vi.fn() }, abort: new AbortController() }
    );
    expect(result.error).toBeDefined();
  });
});
```