# ADR-007: Electron Framework for Desktop Application

## Status
Accepted

## Context

ODEI Symbiosis requires a desktop application that provides:

- **Multi-agent interface:** View and interact with 7 specialized agents (Discuss, Plan, Execute, Mind, Health, Finance, Builder)
- **Graph visualization:** Render Neo4j constitutional graph with 3D/2D interactive views
- **Terminal integration:** Spawn Claude Code processes for each agent, display output
- **Health dashboard:** Real-time biometric data (Apple Watch, Garmin)
- **Calendar sync:** Apple Calendar integration for time blocking
- **Native system access:** File system, clipboard, notifications
- **Cross-platform:** macOS primary, Windows/Linux support
- **Secure IPC:** Renderer cannot directly access backend services
- **Auto-updates:** Deploy new versions without user intervention

### Problem

How do we build a rich desktop UI that safely integrates with ODEI's backend services (Neo4j, MCP servers, system APIs) while maintaining security and cross-platform compatibility?

### Constraints

1. **Security:** Renderer process is untrusted (can load remote content)
2. **Process isolation:** Agent terminals must not freeze UI
3. **Native integration:** Access to filesystem, clipboard, notifications
4. **Development velocity:** Rapid iteration on UI without full rebuilds
5. **Distribution:** Simple packaging for macOS/Windows/Linux
6. **Performance:** Graph with 1000+ nodes must render smoothly

## Decision

**Use Electron as the desktop application framework.**

Electron provides:
1. **Web technologies:** HTML/CSS/JavaScript for UI development
2. **Node.js runtime:** Full access to system APIs, MCP servers
3. **Process model:** Separate main (trusted) and renderer (untrusted) processes
4. **IPC bridge:** Secure communication via `ipcMain`/`ipcRenderer`
5. **Native modules:** Support for `node-pty` (terminals), `better-sqlite3` (history)
6. **Auto-updater:** Built-in update mechanism
7. **Mature ecosystem:** Libraries for graph viz, charts, terminals

### Architecture

```
┌─────────────────────────────────────────────────────────┐
│  Renderer Process (Untrusted)                           │
│  ┌───────────────────────────────────────────────────┐  │
│  │  UI Components (Preact)                           │  │
│  │  - Graph visualization (Sigma.js, 3D-force-graph) │  │
│  │  - Agent terminals (xterm.js)                     │  │
│  │  - Health dashboard (Chart.js)                    │  │
│  │  - Calendar view                                  │  │
│  └───────────────────────────────────────────────────┘  │
│              │                                           │
│              │ IPC messages                              │
│              ▼                                           │
│  ┌───────────────────────────────────────────────────┐  │
│  │  Preload Script (Context Bridge)                  │  │
│  │  - Exposes safe APIs: window.electronAPI         │  │
│  │  - Validates all IPC calls                        │  │
│  └───────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
                       │
                       │ IPC (validated)
                       ▼
┌─────────────────────────────────────────────────────────┐
│  Main Process (Trusted)                                 │
│  ┌───────────────────────────────────────────────────┐  │
│  │  IPC Handlers                                     │  │
│  │  - Agent management (start/stop/output)          │  │
│  │  - MCP server queries (neo4j, history, apple)    │  │
│  │  - System operations (clipboard, file dialogs)   │  │
│  │  - Backup/restore                                 │  │
│  └───────────────────────────────────────────────────┘  │
│              │                                           │
│              ▼                                           │
│  ┌───────────────────────────────────────────────────┐  │
│  │  Backend Services                                 │  │
│  │  - AgentManager (spawn Claude Code processes)    │  │
│  │  - CompanionManager (persistent agent instances) │  │
│  │  - HealthProbes (monitor system health)          │  │
│  │  - BackupManager (Neo4j snapshots)               │  │
│  │  - ConductorServer (agent orchestration)         │  │
│  └───────────────────────────────────────────────────┘  │
│              │                                           │
│              ▼                                           │
│  ┌───────────────────────────────────────────────────┐  │
│  │  External Services                                │  │
│  │  - Neo4j (bolt://127.0.0.1:7687)                 │  │
│  │  - SQLite (data/history.sqlite)                  │  │
│  │  - MCP Servers (stdio processes)                 │  │
│  │  - Body Server (health data)                     │  │
│  └───────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

### IPC Security Model

**Principle:** Renderer process cannot directly call Node.js APIs or access backend services.

**Implementation:**

1. **Context Bridge (Preload Script):**
```javascript
// electron/preload.js
const { contextBridge, ipcRenderer } = require('electron');

contextBridge.exposeInMainWorld('electronAPI', {
  // Safe, validated APIs only
  startAgent: (agentId) => ipcRenderer.invoke('agent:start', agentId),
  queryGraph: (query) => ipcRenderer.invoke('mcp:query', query),
  // Blocked: direct filesystem, shell access
});
```

2. **Main Process Validation:**
```javascript
// electron/ipc-handlers.js
const { validateIPC } = require('./ipc-handlers');

ipcMain.handle('agent:start', async (event, agentId) => {
  // Validate agentId is in allowlist
  if (!['discuss', 'plan', 'execute', 'mind', 'health', 'finance', 'builder'].includes(agentId)) {
    throw new Error('Invalid agent ID');
  }
  return agentManager.start(agentId);
});
```

**Blocked Operations:**
- Direct filesystem access (use dialog APIs instead)
- Shell command execution (use AgentManager APIs)
- Database connections (use MCP tool APIs)

### Agent Lifecycle Management

**AgentManager:**
- Spawns Claude Code processes via `node-pty` (pseudo-TTY for color output)
- Tracks stdout/stderr streams
- Manages process lifecycle (start, stop, restart)
- Provides output to renderer via IPC

**Example:**
```javascript
// Start agent
agentManager.start('discuss');
// Returns: { pid, startedAt, status: 'running' }

// Get output
agentManager.getOutput('discuss', { lines: 50 });
// Returns: { lines: [...], lastSeq: 123 }

// Stop agent
agentManager.stop('discuss');
```

### Terminal Integration (xterm.js)

**Features:**
- Full ANSI color support
- Clipboard integration
- Web links clickable
- Image display (via addon)

**Implementation:**
```javascript
import { Terminal } from '@xterm/xterm';
import { FitAddon } from '@xterm/addon-fit';

const terminal = new Terminal({
  theme: { /* ODEI dark theme */ },
  fontFamily: 'Monaco, monospace',
  fontSize: 13
});

// Receive output from main process
window.electronAPI.onAgentOutput((agentId, lines) => {
  terminal.writeln(lines.join('\n'));
});
```

### Native Modules

**node-pty:**
- Spawn pseudo-terminals (required for Claude Code color output)
- Handles stdin/stdout streams
- Platform-specific build required

**better-sqlite3:**
- Fast SQLite access for history database
- Synchronous API (no callback hell)
- Native addon (requires rebuild after Electron updates)

**Build process:**
```bash
# After npm install or Electron version change
npm run rebuild:native
# Runs: electron-rebuild -f -w better-sqlite3 -w node-pty
```

### Content Security Policy (CSP)

**Policy:**
```html
<meta http-equiv="Content-Security-Policy"
      content="default-src 'self';
               script-src 'self';
               style-src 'self' 'unsafe-inline';
               img-src 'self' data:;
               connect-src 'self'">
```

**Rationale:**
- `'self'` prevents loading external resources
- `'unsafe-inline'` for styles only (required by Tailwind)
- `data:` URIs for inline images (graph viz)
- No `eval()` or remote scripts

### Packaging & Distribution

**electron-builder:**
```json
{
  "build": {
    "appId": "com.odei.app",
    "productName": "ODEI",
    "mac": {
      "category": "public.app-category.productivity",
      "target": [{"target": "dir", "arch": ["arm64"]}],
      "darkModeSupport": true
    },
    "files": [
      "electron/**/*",
      "src/**/*",
      "assets/**/*",
      "node_modules/**/*",
      ".claude/**/*",
      "agents/**/*",
      "servers/**/*"
    ],
    "asarUnpack": [
      "node_modules/node-pty/**/*",
      "servers/**/*",
      ".claude/**/*",
      "agents/**/*"
    ]
  }
}
```

**ASAR Unpacking:**
- Native modules (`node-pty`) cannot be in ASAR archive
- MCP servers must be accessible (spawned as separate processes)
- Agent configurations (`.claude/mcp.json`) must be readable

## Consequences

### Positive

1. **✅ Cross-platform:** Single codebase for macOS/Windows/Linux
2. **✅ Web tech familiarity:** Team knows HTML/CSS/JavaScript
3. **✅ Native integration:** Full access to filesystem, clipboard, notifications
4. **✅ Process isolation:** Agent crashes don't freeze UI
5. **✅ Rich ecosystem:** Libraries for graph viz, charts, terminals
6. **✅ Auto-updates:** electron-updater for seamless deployments
7. **✅ Developer experience:** Hot reload during development

### Negative

1. **⚠️ Large bundle size:** Minimum 150MB (Chromium + Node.js embedded)
2. **⚠️ Memory footprint:** ~200MB base + 100MB per agent terminal
3. **⚠️ Native module complexity:** Requires rebuild after Electron updates
4. **⚠️ Security surface:** Chromium vulnerabilities affect app
5. **⚠️ Startup time:** 2-3 seconds to initialize on first launch
6. **⚠️ Platform quirks:** macOS/Windows/Linux differences in PATH, file dialogs

### Operational Impact

**Development:**
```bash
# Run in dev mode (hot reload)
npm run dev

# Build production app
npm run build:mac
```

**Distribution:**
- macOS: `.app` bundle (~150MB)
- Windows: Installer `.exe` (~180MB)
- Linux: AppImage or `.deb` (~170MB)

**Updates:**
- Auto-update checks on launch
- Downloads delta patches (not full app)
- Requires code signing (Apple notarization for macOS)

**Monitoring:**
- Process memory via Activity Monitor / Task Manager
- IPC message latency tracked in dev tools
- Agent output buffering to prevent memory leaks

## Alternatives Considered

### 1. Web Application (React + Tauri Backend)

**Approach:** React frontend, Rust backend via Tauri

**Pros:**
- Smaller bundle (~40MB)
- Native Rust performance
- Modern web stack

**Cons:**
- Team lacks Rust expertise
- Immature ecosystem vs. Electron
- Harder to integrate Node.js MCP servers
- Limited terminal support

**Rejected:** Electron has proven tooling and Node.js integration.

### 2. Native macOS App (Swift + AppKit)

**Approach:** Pure macOS app in Swift

**Pros:**
- Maximum performance
- Native macOS integration
- Smallest bundle

**Cons:**
- macOS only (no Windows/Linux)
- Team must learn Swift
- No web tech reuse
- Separate implementation per platform

**Rejected:** Cross-platform requirement rules this out.

### 3. Web App + Localhost Server

**Approach:** React SPA, Express backend, browser as UI

**Pros:**
- No packaging complexity
- Standard web development
- Easy updates

**Cons:**
- No native system access
- Browser security restrictions
- Requires managing server lifecycle
- Poor offline experience

**Rejected:** Need native filesystem/clipboard/notifications.

### 4. Qt + QWebEngine

**Approach:** Qt framework with embedded browser

**Pros:**
- Cross-platform
- Native performance
- Mature C++ ecosystem

**Cons:**
- Team must learn C++/Qt
- Licensing costs (commercial use)
- Harder to integrate Node.js
- Steeper learning curve

**Rejected:** Electron's web tech stack matches team skills.

### 5. NW.js (Node-Webkit)

**Approach:** Alternative to Electron, similar architecture

**Pros:**
- Similar to Electron
- Node.js + Chromium
- Smaller learning curve

**Cons:**
- Less mature ecosystem
- Smaller community
- Fewer libraries
- Inconsistent updates

**Rejected:** Electron has better tooling and community support.

## References

- Electron Documentation: https://www.electronjs.org/docs
- Security Best Practices: https://www.electronjs.org/docs/latest/tutorial/security
- electron-builder: https://www.electron.build
- ODEI Electron implementation: `/Users/ai/ODEI/electron/`

## Revision History

- **2025-12-25:** Initial ADR documenting Electron adoption and IPC security model
