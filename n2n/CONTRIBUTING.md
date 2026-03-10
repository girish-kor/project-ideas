# CONTRIBUTING.md — N2N Toolz Contribution Guide

---

## 1. Ways to Contribute

- 🐛 Bug reports
- ✨ Feature requests
- 📝 Documentation improvements
- 🔌 New node types (core)
- 🧩 Plugins (external packages)
- 🧪 Test coverage

---

## 2. Development Setup

```bash
# Fork and clone
git clone https://github.com/YOUR_USERNAME/n2n-toolz.git
cd n2n-toolz

# Install dependencies
npm install

# Start dev mode
npm run dev
```

---

## 3. Branch Naming

| Type | Pattern | Example |
|---|---|---|
| Feature | `feat/short-description` | `feat/kafka-node` |
| Bug fix | `fix/short-description` | `fix/loop-edge-crash` |
| Documentation | `docs/short-description` | `docs/plugin-guide` |
| Refactor | `refactor/short-description` | `refactor/codegen-scope` |
| Test | `test/short-description` | `test/topo-sort-coverage` |

---

## 4. Commit Message Format

Follows [Conventional Commits](https://www.conventionalcommits.org/):

```
<type>(<scope>): <summary>

[optional body]

[optional footer]
```

**Types:** `feat`, `fix`, `docs`, `style`, `refactor`, `test`, `chore`

**Examples:**
```
feat(nodes): add Kafka consumer node
fix(codegen): fix scope variable naming for nested loops
docs(plugin-guide): add testing section
test(runtime): add WorkflowRunner abort test
```

---

## 5. Pull Request Process

1. Create branch from `main`
2. Make changes
3. Run tests: `npm test`
4. Run lint: `npm run lint`
5. Push branch and open PR
6. Fill in PR template
7. Wait for review

**PR Checklist:**
- [ ] Tests added / updated
- [ ] Documentation updated (if applicable)
- [ ] No new lint errors
- [ ] `npm test` passes
- [ ] Added to CHANGELOG.md (for features/fixes)

---

## 6. Adding a New Core Node

### Step 1: Create node definition

```bash
# Create file
touch src/nodes/core/[category]/MyNewNode.ts
```

Follow the full schema in [NODE_SCHEMA.md](./NODE_SCHEMA.md).

### Step 2: Add icon

```bash
# Add SVG icon
cp my-icon.svg src/nodes/icons/my-new-node.svg
```

### Step 3: Register node

```typescript
// src/nodes/index.ts
import { MyNewNode } from './core/[category]/MyNewNode';

NodeRegistry.getInstance().registerAll([
  // existing nodes...
  MyNewNode,
]);
```

### Step 4: Add codegen template (if needed)

```bash
touch src/engine/codegen/templates/myNewNode.ts
```

### Step 5: Write tests

```bash
touch tests/unit/nodes/myNewNode.test.ts
```

### Step 6: Add to NODES.md

Document the node in [NODES.md](./NODES.md) following the existing format.

---

## 7. Code Style

| Rule | Tool |
|---|---|
| Formatting | Prettier (`.prettierrc` config) |
| Linting | ESLint (`tsconfig-eslint` based) |
| Types | TypeScript strict mode |
| Imports | Sorted by type (ESM first, then relative) |

```bash
# Auto-fix
npm run lint:fix
npm run format
```

---

## 8. Testing Requirements

| Change Type | Required Tests |
|---|---|
| New core node | Unit test for `execute()` + code gen snapshot |
| Graph engine change | Unit test with edge cases |
| Code gen change | Snapshot tests for affected node types |
| UI component | Component test with RTL |
| Bug fix | Regression test |
| New plugin | Plugin unit tests (see PLUGIN_GUIDE.md §10) |

**Minimum coverage for PR acceptance:**
- New files: 80% coverage
- Modified files: no coverage regression

---

## 9. Documentation Requirements

| Change | Required Docs |
|---|---|
| New node | Entry in NODES.md |
| New API endpoint | Entry in API.md |
| New env variable | Entry in ENVIRONMENT.md |
| New file format change | Update FILE_FORMAT.md |
| New shortcut | Update SHORTCUTS.md |
| New term | Update GLOSSARY.md |

---

## 10. Issue Reporting

Use GitHub Issues with the correct label:

| Label | Use For |
|---|---|
| `bug` | Something is broken |
| `feature` | New functionality request |
| `docs` | Documentation issue |
| `performance` | Speed/memory issue |
| `security` | Security concern (use email for sensitive issues) |
| `plugin` | Plugin system issue |

**Bug report must include:**
- N2N Toolz version
- OS and version
- Steps to reproduce
- Expected vs actual behavior
- Workflow JSON (if applicable)

---

## 11. Governance

- Core maintainers review all PRs
- Security issues: email `security@n2ntoolz.io`
- Feature discussions: GitHub Discussions
- Community chat: Discord (link in README)
