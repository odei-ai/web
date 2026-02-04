# ODEI Troubleshooting Guide

**Solutions to common issues, error messages, and debugging strategies.**

---

## Table of Contents

1. [Neo4j Connection Issues](#neo4j-connection-issues)
2. [MCP Server Failures](#mcp-server-failures)
3. [Electron Build Problems](#electron-build-problems)
4. [Test Failures](#test-failures)
5. [Agent Terminal Issues](#agent-terminal-issues)
6. [UI Rendering Issues](#ui-rendering-issues)
7. [Performance Problems](#performance-problems)
8. [Common Error Messages](#common-error-messages)

---

## Neo4j Connection Issues

### Error: "ServiceUnavailable: WebSocket connection failure"

**Symptoms:**
- App fails to start
- MCP smoke test fails for odei-neo4j
- Error in logs: `ServiceUnavailable: WebSocket connection failure`

**Diagnosis:**

```bash
# Check if Neo4j is running
nc -zv localhost 7687

# Check Neo4j status in Desktop
# Should show green "Active" indicator
```

**Solutions:**

**1. Start Neo4j Database**
- Open Neo4j Desktop
- Click the "Start" button on your database
- Wait for status to show "Active"

**2. Verify Port 7687 is Open**
```bash
# Check if port is in use
lsof -i :7687

# Should show Neo4j process
# If nothing, Neo4j isn't running
```

**3. Check Database Name**
```bash
# Verify database name in .env
cat .env | grep NEO4J_DATABASE

# Default is "memory"
# If using default Neo4j instance, change to "neo4j"
NEO4J_DATABASE=neo4j
```

**4. Verify Credentials**
```bash
# Test connection with cypher-shell (if installed)
cypher-shell -a bolt://localhost:7687 -u neo4j -p yourpassword

# Or test via Neo4j Browser
# http://localhost:7474
```

**5. Check Firewall**
- macOS: System Preferences → Security & Privacy → Firewall
- Ensure Neo4j Desktop is allowed
- Or disable firewall temporarily to test

### Error: "Authentication failed"

**Symptoms:**
- Error: `Neo.ClientError.Security.Unauthorized`
- Logs show: `Authentication failed`

**Solutions:**

**1. Verify Password in .env**
```bash
# Check password in .env
cat .env | grep NEO4J_PASSWORD

# Should match password set in Neo4j Desktop
```

**2. Reset Neo4j Password**

In Neo4j Desktop:
1. Stop the database
2. Click "..." → Manage → Administration → Set new password
3. Update `.env` with new password
4. Restart database

**3. Check Username**
```bash
# Default username is "neo4j"
NEO4J_USERNAME=neo4j

# Don't change unless you created a different user
```

### Error: "Database 'memory' not found"

**Symptoms:**
- Error: `Neo.DatabaseError.General.UnknownError`
- Logs show: `Database 'memory' does not exist`

**Solutions:**

**1. Create "memory" Database**

In Neo4j Browser (http://localhost:7474):
```cypher
CREATE DATABASE memory;
```

**2. Or Use Default Database**

Edit `.env`:
```bash
NEO4J_DATABASE=neo4j
```

Then restart ODEI.

### Error: "Vector index not found"

**Symptoms:**
- Semantic search fails
- Error: `There is no such index`
- Logs show: `Index not found: goal_embeddings`

**Solutions:**

**1. Install Gen AI Plugin**

In Neo4j Desktop:
1. Select database
2. Click "Plugins" tab
3. Find "Gen AI" plugin
4. Click "Install"
5. Restart database

**2. Recreate Vector Indexes**

```bash
# Run migration script
npm run migrate:vectors

# Or manually in Neo4j Browser:
```

```cypher
CREATE VECTOR INDEX goal_embeddings IF NOT EXISTS
FOR (g:Goal)
ON g.embedding
OPTIONS {indexConfig: {
  `vector.dimensions`: 3072,
  `vector.similarity_function`: 'cosine'
}};
```

---

## MCP Server Failures

### Error: "MCP server failed to start"

**Symptoms:**
- Agent terminal shows: `Error: spawn ENOENT`
- Smoke test fails
- Logs show: `Failed to start MCP server`

**Diagnosis:**

```bash
# Test server directly
cd servers/odei-neo4j
node dist/index.js

# Should wait for input (not crash)
```

**Solutions:**

**1. Rebuild MCP Server**
```bash
npm run build

# Or rebuild specific server
cd servers/odei-neo4j
npm run build
cd ../..
```

**2. Check TypeScript Compilation**
```bash
npm run typecheck

# Fix any type errors shown
```

**3. Verify Dependencies**
```bash
# Reinstall dependencies
npm install

# In server directory
cd servers/odei-neo4j
npm install
cd ../..
```

**4. Check Node Version**
```bash
node --version
# Should be 22+

# Update if needed
nvm install 22
nvm use 22
```

### Error: "Cannot find module '@modelcontextprotocol/sdk'"

**Symptoms:**
- MCP server crashes on start
- Error: `Cannot find module '@modelcontextprotocol/sdk'`

**Solutions:**

**1. Install MCP SDK**
```bash
cd servers/odei-neo4j
npm install @modelcontextprotocol/sdk
cd ../..
```

**2. Reinstall All Dependencies**
```bash
npm run clean
npm install
npm run build
```

### Error: "OpenAI API key not found"

**Symptoms:**
- Bootstrap fails
- Semantic search fails
- Error: `OPENAI_API_KEY is required`

**Solutions:**

**1. Add API Key to .env**
```bash
# Edit .env
nano .env

# Add:
OPENAI_API_KEY=sk-proj-your_key_here
```

**2. Verify Environment Variable is Loaded**
```bash
# In Node.js
node -e "require('dotenv').config(); console.log(process.env.OPENAI_API_KEY)"

# Should print your key (or first few characters)
```

**3. Restart ODEI** (environment variables loaded at startup)

### MCP Smoke Test Failures

**Run diagnostics:**
```bash
npm run smoke
```

**Interpret results:**
```
✓ odei-history: OK          # ✅ Working
✓ odei-neo4j: OK            # ✅ Working
✗ odei-apple: FAIL          # ❌ Not working (macOS only)
✓ odei-gemini: OK           # ✅ Working
✗ odei-telegram: FAIL       # ❌ Not configured
```

**Fix failing servers:**

**odei-apple (macOS only):**
```bash
cd servers/odei-apple
swift build -c release
cd ../..
```

**odei-telegram:**
```bash
# Add to .env
TELEGRAM_BOT_TOKEN=your_token_here
TELEGRAM_ALLOWED_CHAT_IDS=123456789
```

---

## Electron Build Problems

### Error: "Module did not self-register"

**Symptoms:**
- Electron crashes on start
- Error: `Error: Module did not self-register`
- Related to native modules (node-pty, better-sqlite3, etc.)

**Solutions:**

**1. Rebuild Native Modules**
```bash
npm run rebuild:native
```

**2. Rebuild Specific Module**
```bash
./node_modules/.bin/electron-rebuild -m node-pty
```

**3. Clean and Reinstall**
```bash
npm run clean
npm install
npm run rebuild:native
```

**4. Check Electron Version**
```bash
# Verify Electron version matches
npm list electron

# Should match version in package.json
```

### Error: "Cannot find module 'electron'"

**Symptoms:**
- `npm start` fails
- Error: `Cannot find module 'electron'`

**Solutions:**

**1. Install Electron**
```bash
npm install electron
```

**2. Run postinstall**
```bash
npm run postinstall
```

### Error: "Application not responding"

**Symptoms:**
- Electron hangs on startup
- Window opens but is blank
- No console logs

**Solutions:**

**1. Check Main Process Logs**
```bash
# Run with logging
npm run dev:electron

# Look for errors in terminal
```

**2. Disable Agent Spawning** (temporary)

Edit `electron/main.js`:
```javascript
// Comment out agent spawning
// await spawnAgents();
```

**3. Check for Infinite Loops**

Common causes:
- IPC handler calling itself
- Event listener triggering same event
- Recursive function without base case

### Build Fails with "EACCES: permission denied"

**Symptoms:**
- `npm run build:mac` fails
- Error: `EACCES: permission denied`

**Solutions:**

**1. Fix Script Permissions**
```bash
chmod +x scripts/*.sh
```

**2. Clean Build Directory**
```bash
rm -rf dist-electron
npm run build:mac
```

**3. Run with Sudo** (not recommended)
```bash
sudo npm run build:mac
```

---

## Test Failures

### Playwright Tests Fail

**Symptoms:**
- `npm run test:playwright` fails
- Browser doesn't launch
- Tests timeout

**Solutions:**

**1. Install Playwright Browsers**
```bash
npx playwright install
```

**2. Run in Headed Mode** (see what's happening)
```bash
npx playwright test --headed
```

**3. Increase Timeout**

Edit `playwright.config.ts`:
```typescript
export default defineConfig({
  timeout: 60000, // Increase from 30s to 60s
});
```

**4. Check Electron Path**

Edit `tests/example.spec.ts`:
```typescript
import { _electron as electron } from 'playwright';
import path from 'path';

const electronPath = path.join(__dirname, '../node_modules/.bin/electron');
console.log('Electron path:', electronPath);
```

### Vitest Tests Fail

**Symptoms:**
- `npm run test:unit` fails
- Import errors
- Module not found

**Solutions:**

**1. Check Import Paths**
```javascript
// Use .js extension for imports
import { myFunction } from './myModule.js'; // ✅
import { myFunction } from './myModule';    // ❌
```

**2. Clear Vitest Cache**
```bash
rm -rf node_modules/.vitest
npm run test:unit
```

**3. Check vitest.config.ts**
```typescript
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    globals: true,
    environment: 'jsdom', // For DOM tests
  },
});
```

---

## Agent Terminal Issues

### Agent Won't Start

**Symptoms:**
- Agent tab shows "Starting..."
- Terminal never appears
- No output

**Diagnosis:**

```bash
# Check agent logs
cat logs/discuss.log

# Or check main process logs
npm run dev:electron
```

**Solutions:**

**1. Verify MCP Config**
```bash
cd agents/discuss
cat .claude/mcp.json

# Validate JSON syntax
cat .claude/mcp.json | python3 -m json.tool
```

**2. Check MCP Server Paths**
```json
{
  "mcpServers": {
    "odei-neo4j": {
      "command": "node",
      "args": ["../../servers/odei-neo4j/dist/index.js"],
      // ^^^ Verify this path is correct
    }
  }
}
```

**3. Rebuild MCP Servers**
```bash
npm run build
```

**4. Check Environment Variables**
```json
{
  "env": {
    "NEO4J_PASSWORD": "${NEO4J_PASSWORD}"
    // ^^^ Verify this is defined in root .env
  }
}
```

### Agent Output Not Showing

**Symptoms:**
- Agent terminal blank
- No text appears when typing
- Commands don't execute

**Solutions:**

**1. Check PTY Process**
```bash
# View running processes
ps aux | grep node-pty

# Should show agent processes
```

**2. Restart Agent**
- Click "Restart" button in agent tab
- Or restart entire ODEI app

**3. Check IPC Communication**

Open DevTools (Cmd+Option+I) → Console:
```javascript
// Test IPC
window.api.terminalInput({ agentId: 'discuss', input: 'echo test\n' });
```

### Agent Crashes on Start

**Symptoms:**
- Agent starts then immediately stops
- Error in logs
- Tab shows "Error"

**Diagnosis:**

```bash
# Run agent manually
cd agents/discuss
claude .

# Look for errors
```

**Common Causes:**

**1. MCP Server Not Found**
```bash
# Verify server exists
ls -la ../../servers/odei-neo4j/dist/index.js
```

**2. Invalid MCP Config**
```bash
# Validate JSON
cat .claude/mcp.json | python3 -m json.tool
```

**3. Missing Environment Variables**
```bash
# Check .env exists in root
ls -la ../../.env
```

---

## UI Rendering Issues

### CSS Not Updating

**Symptoms:**
- Changed Tailwind classes don't take effect
- Styles look wrong
- Old styles persist

**Solutions:**

**1. Rebuild CSS**
```bash
npm run build:css
```

**2. Hard Refresh**
- macOS: `Cmd+Shift+R`
- Windows/Linux: `Ctrl+Shift+R`

**3. Clear Cache**

Open DevTools → Console:
```javascript
localStorage.clear();
sessionStorage.clear();
location.reload(true);
```

**4. Check Tailwind Config**
```bash
# Verify tailwind.config.js exists
cat tailwind.config.js

# Should have content paths:
# content: ['./src/**/*.html', './src/**/*.js']
```

### Graph Visualization Not Loading

**Symptoms:**
- Memory Atlas tab is blank
- No graph visible
- Console errors

**Diagnosis:**

Open DevTools → Console:
- Look for errors
- Check Network tab for failed requests

**Solutions:**

**1. Check Data Loading**

DevTools → Console:
```javascript
// Test graph data fetch
window.api.getGraphData().then(data => console.log(data));
```

**2. Verify Sigma.js**
```bash
# Check if sigma is installed
npm list sigma

# Reinstall if needed
npm install sigma graphology
```

**3. Check Canvas Rendering**

DevTools → Console:
```javascript
// Test canvas
const canvas = document.querySelector('canvas');
console.log(canvas); // Should not be null
```

### Terminal UI Issues

**Symptoms:**
- xterm.js terminal not rendering
- Text garbled or missing
- Colors wrong

**Solutions:**

**1. Verify xterm.js Installation**
```bash
npm list @xterm/xterm
npm list @xterm/addon-fit
```

**2. Check Terminal Initialization**

DevTools → Console:
```javascript
// Should see xterm objects
console.log(window.terminals);
```

**3. Reload Addons**
```javascript
// In renderer.js
import { FitAddon } from '@xterm/addon-fit';
const fitAddon = new FitAddon();
term.loadAddon(fitAddon);
```

---

## Performance Problems

### Slow Startup

**Symptoms:**
- ODEI takes >10 seconds to start
- White screen for extended period
- Fans spin up

**Diagnosis:**

```bash
# Check startup time
time npm start
```

**Solutions:**

**1. Optimize MCP Server Loading**

Edit `.claude/mcp.json`:
```json
{
  "mcpServers": {
    // Only load servers you need
    "odei-neo4j": { ... },
    "odei-history": { ... }
    // Remove unused servers
  }
}
```

**2. Lazy Load Heavy Dependencies**
```javascript
// Instead of import at top
import { Chart } from 'chart.js';

// Import when needed
async function showChart() {
  const { Chart } = await import('chart.js');
  // ...
}
```

**3. Check Neo4j Connection**
- Slow connection → Check network/firewall
- Large database → Consider pagination

### High Memory Usage

**Symptoms:**
- ODEI uses >2GB RAM
- System slows down
- Out of memory errors

**Diagnosis:**

Open DevTools → Memory tab:
- Take heap snapshot
- Look for large objects
- Check for memory leaks

**Solutions:**

**1. Limit Graph Data**
```javascript
// Don't load entire graph
// Instead, paginate:
const nodes = await getNodes({ limit: 100 });
```

**2. Clean Up Event Listeners**
```javascript
// Always remove listeners
window.addEventListener('resize', handler);
// Later:
window.removeEventListener('resize', handler);
```

**3. Dispose Charts Properly**
```javascript
// When removing chart
chart.destroy();
```

### Slow Graph Queries

**Symptoms:**
- Semantic search takes >5 seconds
- Graph visualization loads slowly
- Neo4j queries timeout

**Diagnosis:**

In Neo4j Browser:
```cypher
PROFILE MATCH (g:Goal)
WHERE g.embedding IS NOT NULL
RETURN g
LIMIT 10;
```

Look for "Db Hits" in profile output.

**Solutions:**

**1. Create Indexes**
```cypher
CREATE INDEX goal_title FOR (g:Goal) ON (g.title);
CREATE FULLTEXT INDEX goal_search FOR (g:Goal) ON EACH [g.title, g.summary];
```

**2. Optimize Queries**
```cypher
// ❌ Bad: Scans all nodes
MATCH (g:Goal)
WHERE g.title CONTAINS 'test'
RETURN g;

// ✅ Good: Uses index
MATCH (g:Goal)
WHERE g.title STARTS WITH 'test'
RETURN g;
```

**3. Limit Results**
```cypher
MATCH (g:Goal)
RETURN g
LIMIT 100; // Always limit
```

---

## Common Error Messages

### "EADDRINUSE: Address already in use"

**Meaning:** Port is already occupied (usually Neo4j or Body Server)

**Solution:**
```bash
# Find process using port 7687 (Neo4j)
lsof -i :7687

# Kill process
kill -9 <PID>

# Or change port in .env
NEO4J_URI=bolt://localhost:7688
```

### "ENOENT: no such file or directory"

**Meaning:** File path is wrong

**Solution:**
```bash
# Verify file exists
ls -la path/to/file

# Check working directory
pwd

# Use absolute paths in code
const filePath = path.join(__dirname, 'relative/path');
```

### "SyntaxError: Unexpected token"

**Meaning:** JavaScript syntax error (often JSON parsing)

**Solution:**
```javascript
// Validate JSON
try {
  JSON.parse(data);
} catch (err) {
  console.error('Invalid JSON:', err.message);
}

// Or validate with tool
cat file.json | python3 -m json.tool
```

### "TypeError: Cannot read property 'X' of undefined"

**Meaning:** Trying to access property on undefined/null

**Solution:**
```javascript
// Use optional chaining
const value = obj?.property?.nested;

// Or check first
if (obj && obj.property) {
  const value = obj.property.nested;
}
```

### "Error: Timeout exceeded"

**Meaning:** Operation took too long

**Solution:**
```javascript
// Increase timeout
const result = await operation({ timeout: 60000 }); // 60s

// Or optimize operation
// - Add indexes
// - Limit results
// - Use pagination
```

---

## Getting More Help

### Collect Diagnostic Information

Before asking for help, gather:

**1. System Info**
```bash
node --version
npm --version
electron --version  # or check package.json
neo4j version       # in Neo4j Desktop
```

**2. Error Logs**
```bash
# Main process logs
cat logs/main.log

# Agent logs
cat logs/discuss.log

# MCP server logs (if available)
```

**3. Configuration**
```bash
# Anonymize passwords first!
cat .env
cat .claude/mcp.json
```

**4. Recent Changes**
```bash
git log --oneline -10
```

### Enable Verbose Logging

**Electron:**
```bash
LOG_LEVEL=debug npm run dev
```

**MCP Servers:**
```json
{
  "env": {
    "LOG_LEVEL": "debug"
  }
}
```

### Report Issues

When reporting issues, include:
1. OS and version
2. Node.js version
3. Neo4j version
4. Steps to reproduce
5. Expected vs actual behavior
6. Error messages (full stack trace)
7. Logs (sanitized)

---

## Emergency Recovery

### Nuclear Option: Complete Reset

**Warning:** This will delete all data and configurations.

```bash
# 1. Stop ODEI and Neo4j
killall electron
# Stop Neo4j in Desktop

# 2. Backup Neo4j data
npm run backup:neo4j

# 3. Delete everything
npm run clean
rm -rf node_modules
rm -rf servers/*/node_modules
rm -rf servers/*/dist

# 4. Reset Neo4j database
# In Neo4j Desktop: Stop → Delete Database → Recreate

# 5. Reinstall from scratch
npm install
npm run setup

# 6. Reconfigure .env
cp .env.example .env
nano .env

# 7. Bootstrap foundation
npm run odei:bootstrap

# 8. Restart
npm run dev
```

---

_Last updated: 2025-12-25_
