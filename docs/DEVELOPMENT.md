# ODEI Development Guide

**Day-to-day development workflow, best practices, and contribution guidelines.**

---

## Table of Contents

1. [Development Workflow](#development-workflow)
2. [Running in Dev Mode](#running-in-dev-mode)
3. [Making Changes](#making-changes)
4. [Testing](#testing)
5. [Code Style Guide](#code-style-guide)
6. [Git Workflow](#git-workflow)
7. [Common Development Tasks](#common-development-tasks)

---

## Development Workflow

### Daily Routine

```bash
# 1. Pull latest changes
git pull origin main

# 2. Install any new dependencies
npm install

# 3. Rebuild if dependencies changed
npm run build

# 4. Start development mode
npm run dev

# 5. Make changes
# 6. Test changes
npm run test:unit

# 7. Commit and push
git add .
git commit -m "feat(ui): Add feature X"
git push origin main
```

### Two-Workspace Model

ODEI has **two types of workspaces**:

#### 1. Root Workspace (Infrastructure Development)

**Location:** `/Users/ai/ODEI/`

**Use for:**
- Electron main/renderer process work
- UI components (HTML, CSS, JavaScript)
- MCP server development (`servers/*`)
- Build configuration
- General project structure

**MCP Servers:**
- odei-neo4j (Memory Atlas)
- odei-history (Conversation persistence)

**Start workspace:**
```bash
cd /Users/ai/ODEI
claude .
```

#### 2. Agent Workspaces (Agent Development)

**Location:** `agents/discuss/`, `agents/plan/`, etc.

**Use for:**
- Agent-specific logic
- Agent MCP configuration
- Agent automation scripts
- Testing agent protocols

**MCP Servers:** Each agent has its own `.claude/mcp.json`

**Start workspace:**
```bash
cd agents/discuss
claude .
```

**Rule of Thumb:**
- Editing `electron/` or `src/` → Use **root workspace**
- Editing `agents/*/` → Use **agent workspace**

---

## Running in Dev Mode

### Method 1: Full Development Mode (Recommended)

```bash
npm run dev
```

This runs:
- Electron with logging enabled
- CSS watch mode (auto-rebuild on changes)
- Node environment set to development

**Benefits:**
- Hot reload for CSS changes
- Full console logging
- DevTools enabled

### Method 2: CSS Watch Only

```bash
# Terminal 1: Watch CSS
npm run dev:css

# Terminal 2: Run Electron
npm run dev:electron
```

**Use when:** You want separate control over CSS and Electron.

### Method 3: Production Mode

```bash
npm start
```

**Use when:** Testing production-like behavior (no logging, no hot reload).

### Opening DevTools

**Keyboard Shortcut:**
- macOS: `Cmd+Option+I`
- Windows/Linux: `Ctrl+Shift+I`

**Or add to code:**
```javascript
// electron/main.js
mainWindow.webContents.openDevTools();
```

---

## Making Changes

### UI Changes

#### Editing HTML

**File:** `src/index.html` (main page) or `src/memory.html` (Memory Atlas)

**After changes:**
- No rebuild needed
- Refresh Electron: `Cmd+R` (macOS) or `Ctrl+R` (Windows/Linux)

#### Editing CSS

**Files:**
- `src/input.css` — Tailwind input
- `src/styles.css` — Generated output (don't edit directly)

**After changes:**
```bash
npm run build:css
# Or use watch mode: npm run dev:css
```

**Important:** Always edit `input.css`, not `styles.css`.

#### Editing JavaScript (Renderer)

**Files:** `src/*.js` (renderer process)

**After changes:**
- No rebuild needed
- Refresh Electron: `Cmd+R` (macOS) or `Ctrl+R` (Windows/Linux)

### Electron Main Process Changes

**Files:** `electron/*.js`

**After changes:**
- Restart Electron completely (Cmd+Q or Ctrl+Q, then `npm run dev`)

**Why:** Main process doesn't support hot reload.

### MCP Server Changes

**Files:** `servers/*/src/*.ts`

**After changes:**
```bash
# Rebuild specific server
cd servers/odei-neo4j
npm run build
cd ../..

# Or rebuild all servers
npm run build
```

**Then:** Restart ODEI (MCP servers loaded on app start).

### Agent Configuration Changes

**Files:** `agents/*/.claude/mcp.json`, `agents/*/.claude/CLAUDE.md`

**After changes:**
- Restart agent terminal (click "Restart" in UI or restart ODEI)

---

## Testing

### Unit Tests (Vitest)

**Run all tests:**
```bash
npm run test:unit
```

**Run in watch mode:**
```bash
npm run test:watch
```

**Run with UI:**
```bash
npm run test:ui
```

**Run with coverage:**
```bash
npm run test:coverage
```

**Writing a test:**

```javascript
// tests/example.test.js
import { describe, it, expect } from 'vitest';
import { myFunction } from '../src/myModule.js';

describe('myFunction', () => {
  it('should return expected value', () => {
    expect(myFunction(1, 2)).toBe(3);
  });
});
```

### E2E Tests (Playwright)

**Run E2E tests:**
```bash
npm run test:playwright
```

**Run in headed mode (see browser):**
```bash
npx playwright test --headed
```

**Debug a test:**
```bash
npx playwright test --debug
```

**Writing an E2E test:**

```typescript
// tests/example.spec.ts
import { test, expect } from '@playwright/test';

test('should load app', async ({ page }) => {
  await page.goto('file:///path/to/index.html');
  await expect(page.locator('h1')).toContainText('ODEI');
});
```

### MCP Server Tests

**Smoke test all servers:**
```bash
npm run smoke
```

**Test specific server:**
```bash
# Start server manually and test
cd servers/odei-neo4j
npm run build
node dist/index.js
```

### Testing Checklist

Before committing:
- [ ] Unit tests pass: `npm run test:unit`
- [ ] E2E tests pass: `npm run test:playwright`
- [ ] MCP servers healthy: `npm run smoke`
- [ ] TypeScript compiles: `npm run typecheck`
- [ ] Linter passes: `npm run lint`
- [ ] App starts: `npm run dev`

---

## Code Style Guide

### General Principles

1. **Read before writing** — Never edit code you haven't read
2. **Follow existing patterns** — Match the codebase style
3. **Prefer clarity over cleverness** — Code is read more than written
4. **Comment the "why", not the "what"** — Code explains what, comments explain why

### JavaScript/TypeScript

**Style:**
- Use ES6+ features (const/let, arrow functions, destructuring)
- Prefer `const` over `let`, never use `var`
- Use template literals for string interpolation
- Use async/await over callbacks

**Good:**
```javascript
const getUserName = async (userId) => {
  const user = await db.getUser(userId);
  return user?.name ?? 'Unknown';
};
```

**Bad:**
```javascript
var getUserName = function(userId) {
  return db.getUser(userId).then(function(user) {
    if (user && user.name) {
      return user.name;
    } else {
      return 'Unknown';
    }
  });
};
```

**TypeScript:**
- Use strict mode
- Avoid `any` (use `unknown` if needed)
- Prefer interfaces over types for objects
- Use type guards for narrowing

**Good:**
```typescript
interface User {
  id: string;
  name: string;
  email?: string;
}

function isUser(obj: unknown): obj is User {
  return typeof obj === 'object' && obj !== null && 'id' in obj;
}
```

### File Organization

**Electron:**
```
electron/
├── main.js              # Entry point
├── agent-manager.js     # Agent lifecycle
├── ipc/                 # IPC handlers (organized by domain)
│   ├── terminal.js
│   ├── memory.js
│   └── health.js
└── preload.js           # Context bridge
```

**Renderer:**
```
src/
├── renderer.js          # Main renderer logic
├── modules/             # Feature modules
│   ├── conversation-manager.js
│   ├── memory-visualization.js
│   └── health-dashboard.js
└── utils/               # Shared utilities
    ├── format.js
    └── validation.js
```

### Naming Conventions

**Files:** kebab-case
```
conversation-manager.js ✅
conversationManager.js ❌
ConversationManager.js ❌
```

**Functions/Variables:** camelCase
```javascript
const userName = 'Alice';        // ✅
const UserName = 'Alice';        // ❌
const user_name = 'Alice';       // ❌

function getUserName() {}        // ✅
function get_user_name() {}      // ❌
function GetUserName() {}        // ❌
```

**Classes:** PascalCase
```javascript
class UserManager {}             // ✅
class userManager {}             // ❌
class user_manager {}            // ❌
```

**Constants:** UPPER_SNAKE_CASE (only for true constants)
```javascript
const MAX_RETRIES = 3;           // ✅
const API_BASE_URL = '...';      // ✅
const userName = 'Alice';        // ✅ (not a constant)
const USER_NAME = 'Alice';       // ❌ (not a constant)
```

### UI Styling (Tailwind)

**Use design system tokens:**
```html
<!-- ✅ Correct: Use teal (jade) accent -->
<button class="bg-teal-500 hover:bg-teal-400 text-white">
  Click me
</button>

<!-- ❌ Wrong: Don't use indigo/purple/violet -->
<button class="bg-indigo-500">
  Click me
</button>
```

**Follow button hierarchy:**
```html
<!-- Primary CTA: Gradient button -->
<button class="primary-action-btn">
  Save Changes
</button>

<!-- Secondary: Subtle gradient -->
<button class="conv-action-btn">
  Go to Thread
</button>

<!-- Glass: Transparent with border -->
<button class="bg-white/5 border border-teal-500/30">
  Toggle View
</button>
```

**Read the design spec:**
- Always check `docs/premium-dark-theme.md` before making UI changes
- Use color tokens from the spec
- Maintain visual hierarchy

### Error Handling

**Always handle errors explicitly:**
```javascript
// ✅ Good
try {
  const result = await riskyOperation();
  return result;
} catch (error) {
  console.error('Operation failed:', error);
  return { error: error.message };
}

// ❌ Bad (silent failure)
const result = await riskyOperation().catch(() => null);
```

**Use Zod for validation:**
```typescript
import { z } from 'zod';

const UserSchema = z.object({
  id: z.string(),
  name: z.string(),
  email: z.string().email().optional()
});

// Validate before use
const user = UserSchema.parse(untrustedData);
```

---

## Git Workflow

### Commit Messages

**Format:** `<type>(<scope>): <subject>`

**Types:**
- `feat` — New feature
- `fix` — Bug fix
- `refactor` — Code refactoring (no functional change)
- `docs` — Documentation changes
- `style` — Code style changes (formatting, etc.)
- `test` — Add/update tests
- `chore` — Build config, dependencies, etc.

**Scopes:**
- `ui` — Renderer process, HTML, CSS
- `electron` — Main process
- `neo4j` — odei-neo4j server
- `history` — odei-history server
- `agents` — Agent configuration/logic
- `build` — Build system, dependencies

**Examples:**
```bash
git commit -m "feat(ui): Add health dashboard vitals chart"
git commit -m "fix(neo4j): Resolve embedding generation timeout"
git commit -m "refactor(agents): Simplify discuss agent protocol"
git commit -m "docs(onboarding): Add troubleshooting section"
git commit -m "chore(deps): Update Electron to 38.2.2"
```

### Branch Strategy

**Main branch:** `main`
- Always stable
- Deploy from main

**Feature branches:** Optional for large changes
```bash
git checkout -b feat/health-dashboard
# ... make changes ...
git commit -m "feat(ui): Add health dashboard"
git push origin feat/health-dashboard
# ... create PR ...
```

**For small changes:** Commit directly to main
```bash
git add .
git commit -m "fix(ui): Correct typo in button text"
git push origin main
```

### Pre-commit Hooks

**Husky + lint-staged** configured to run:
- ESLint on `.js`, `.ts` files (when configured)
- Prettier on `.css`, `.md`, `.json` files

**Skip hooks (not recommended):**
```bash
git commit --no-verify -m "..."
```

---

## Common Development Tasks

### Add a Dependency

```bash
# Production dependency
npm install <package>

# Development dependency
npm install -D <package>

# Native module (needs rebuild)
npm install <native-package>
npm run rebuild:native
```

### Update Dependencies

```bash
# Check for updates
npm run update:check

# Update patch/minor versions (safe)
npm run update:safe

# Update all (including major, breaking)
npm run update:all
```

### Create a New MCP Server

```bash
# 1. Create server directory
mkdir servers/odei-myserver
cd servers/odei-myserver

# 2. Initialize npm
npm init -y

# 3. Install MCP SDK
npm install @modelcontextprotocol/sdk

# 4. Create src directory
mkdir src
touch src/index.ts

# 5. Add tsconfig.json
cat > tsconfig.json << EOF
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "commonjs",
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true
  }
}
EOF

# 6. Add build script to package.json
# "scripts": { "build": "tsc" }

# 7. Implement server in src/index.ts

# 8. Build
npm run build

# 9. Test
node dist/index.js
```

### Add a New Agent

```bash
# 1. Create agent directory
mkdir agents/myagent
cd agents/myagent

# 2. Create .claude directory
mkdir .claude

# 3. Create mcp.json
cat > .claude/mcp.json << EOF
{
  "mcpServers": {
    "odei-neo4j": {
      "command": "node",
      "args": ["../../servers/odei-neo4j/dist/index.js"],
      "env": {
        "NEO4J_URI": "bolt://localhost:7687",
        "NEO4J_USERNAME": "neo4j",
        "NEO4J_PASSWORD": "${NEO4J_PASSWORD}"
      }
    }
  }
}
EOF

# 4. Create CLAUDE.md with agent instructions

# 5. Update electron/agent-manager.js to include new agent

# 6. Restart ODEI
```

### Add a Graph Node Type

```bash
# 1. Edit schema in odei-neo4j
cd servers/odei-neo4j
nano src/domain/schemaInventory.ts

# 2. Add node type definition

# 3. Implement create/update tools

# 4. Build server
npm run build

# 5. Export updated schema
npm run schema:export

# 6. Test with smoke test
cd ../..
npm run smoke
```

### Regenerate Embeddings

```bash
# Regenerate all embeddings (useful after changing embedding model)
npm run odei:regenerate-embeddings

# Test embedding quality
npm run odei:test-embeddings
```

### Backup Neo4j Database

```bash
# Manual backup
npm run backup:neo4j

# Backups stored in: backups/neo4j-YYYY-MM-DD-HHMMSS.json
```

### Clean Build Artifacts

```bash
# Remove all build artifacts and node_modules
npm run clean

# Then reinstall and rebuild
npm run setup
```

### Fix Native Module Issues

```bash
# Rebuild all native modules for current Electron version
npm run rebuild:native

# Or rebuild specific module
./node_modules/.bin/electron-rebuild -m <module-name>
```

---

## Debugging

### Debugging Main Process

**Method 1: Console logs**
```javascript
// electron/main.js
console.log('Debug info:', data);
```

Logs appear in terminal where you ran `npm run dev`.

**Method 2: VSCode debugger**

Create `.vscode/launch.json`:
```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Debug Electron Main",
      "type": "node",
      "request": "launch",
      "cwd": "${workspaceFolder}",
      "runtimeExecutable": "${workspaceFolder}/node_modules/.bin/electron",
      "windows": {
        "runtimeExecutable": "${workspaceFolder}/node_modules/.bin/electron.cmd"
      },
      "args": ["."],
      "outputCapture": "std"
    }
  ]
}
```

Then press F5 in VSCode.

### Debugging Renderer Process

**Open DevTools:** `Cmd+Option+I` (macOS) or `Ctrl+Shift+I` (Windows/Linux)

**Console tab:** View logs
```javascript
// src/renderer.js
console.log('Debug info:', data);
```

**Sources tab:** Set breakpoints
1. Find your file in Sources
2. Click line number to set breakpoint
3. Trigger code (click button, etc.)
4. Inspector pauses at breakpoint

**Network tab:** Monitor IPC calls (not HTTP, but shows resource loading)

### Debugging MCP Servers

**Method 1: Direct execution**
```bash
cd servers/odei-neo4j
npm run build
node dist/index.js
```

Server will wait for stdio input. Send JSON-RPC manually to test.

**Method 2: Logs**

Add logging in server code:
```typescript
// servers/odei-neo4j/src/index.ts
console.error('Debug:', data);  // Logs to stderr
```

Stderr visible in terminal.

### Common Debugging Scenarios

**App won't start:**
1. Check terminal for errors
2. Verify Neo4j is running: `nc -zv localhost 7687`
3. Check `.env` credentials
4. Try `npm run rebuild:native`

**Agent won't start:**
1. Check `.claude/mcp.json` syntax
2. Verify MCP servers are built
3. Check environment variables
4. Restart ODEI

**UI not updating:**
1. Hard refresh: `Cmd+Shift+R` (macOS) or `Ctrl+Shift+R` (Windows/Linux)
2. Rebuild CSS: `npm run build:css`
3. Clear cache: `localStorage.clear()` in DevTools Console

---

## Best Practices

### 1. Always Read Before Editing

Never make changes to code you haven't read. Understand the context first.

### 2. Match Existing Patterns

Look at similar code in the codebase and follow the same patterns.

### 3. Write Tests for New Features

Every new feature should have:
- Unit tests for logic
- E2E tests for UI interactions

### 4. Validate All IPC with Zod

```javascript
// electron/ipc/example.js
import { z } from 'zod';

const MyRequestSchema = z.object({
  userId: z.string(),
  action: z.enum(['create', 'update', 'delete'])
});

ipcMain.handle('my-request', async (event, data) => {
  const validated = MyRequestSchema.parse(data);
  // ... safe to use validated data
});
```

### 5. Use Semantic Versioning

When updating dependencies, follow semver:
- **Patch (1.0.x):** Bug fixes, safe to update
- **Minor (1.x.0):** New features, backward compatible
- **Major (x.0.0):** Breaking changes, review carefully

### 6. Keep MCP Servers Stateless

MCP servers should not store state between requests. All state in database.

### 7. Document Complex Logic

If code is hard to understand, add comments explaining the "why":

```javascript
// Use cosine similarity instead of dot product because embeddings
// are normalized to unit length, making cosine faster and equivalent
const similarity = cosineSimilarity(a, b);
```

---

## Performance Tips

### 1. Lazy Load Heavy Dependencies

```javascript
// ❌ Bad: Import at top (always loaded)
import { Chart } from 'chart.js';

// ✅ Good: Import when needed
async function showChart() {
  const { Chart } = await import('chart.js');
  // ... use Chart
}
```

### 2. Debounce Expensive Operations

```javascript
// Debounce search to avoid hammering database
const debouncedSearch = debounce(async (query) => {
  const results = await searchGraph(query);
  updateUI(results);
}, 300);
```

### 3. Batch Graph Queries

```javascript
// ❌ Bad: N+1 queries
for (const nodeId of nodeIds) {
  const node = await getNode(nodeId);
}

// ✅ Good: Single batch query
const nodes = await getNodes(nodeIds);
```

### 4. Use Indexes

All searchable properties should have indexes:
```cypher
CREATE INDEX goal_title FOR (g:Goal) ON (g.title);
CREATE FULLTEXT INDEX goal_search FOR (g:Goal) ON EACH [g.title, g.summary];
```

---

_Last updated: 2025-12-25_
