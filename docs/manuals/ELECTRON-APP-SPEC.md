# ODEI Electron Application — Functional Specification

**Version**: 1.0
**Status**: Ready for Implementation (Phase 2)
**Date**: 2025-10-09

---

## Executive Summary

The ODEI Electron application is a production-ready desktop interface for the 4-agent ODEI system. It provides:

1. **4 PTY Terminals** — Real Claude Code instances running in isolated workspaces
2. **Unified Dashboard** — Single window with agent tabs, health monitoring, and data panels
3. **Control Frame Protocol** — Automatic agent handoffs and system coordination
4. **MCP Integration** — All 3 MCP servers (Neo4j, History, Apple) connected and monitored

**Key Principle**: The Electron app is an **orchestrator**, not a controller. Agents run independently, and the app provides visibility, coordination, and user interaction.

---

## 🎨 UI/UX Design (Based on Frontend Prototype)

### Layout Structure

```
┌─────────────────────────────────────────────────────────────┐
│  [ODEI Logo] [Discuss] [Plan] [Execute] [Mind] [Body] [Finance] [Builder] │  ← Navbar
├─────────────────────────────────────────┬───────────────────┤
│                                         │                   │
│                                         │  Data Panel       │
│         Terminal View                   │  ───────────────  │
│         (xterm.js)                      │  • Agent Status   │
│                                         │  • Control Frames │
│                                         │  • Recent Events  │
│                                         │                   │
├─────────────────────────────────────────┴───────────────────┤
│  Status: discuss • Agent: Running • Neo4j: ✅ • MCP: ✅      │  ← Status Bar
└─────────────────────────────────────────────────────────────┘
```

### Visual Design System

#### Colors (Dark Theme)

```css
Background:    #0a0a0f (gray-950)
Surface:       #1f2937 (gray-900)
Border:        #374151 (gray-700)
Text Primary:  #E3E4E8 (gray-100)
Text Secondary:#9CA3AF (gray-400)
Accent:        #5E6AD2 (indigo-500)
Success:       #10B981 (green-500)
Error:         #EF4444 (red-500)
Warning:       #F59E0B (amber-500)
```

#### Typography

```css
Font Family: 'Inter', system-ui, sans-serif
Terminal:    'Menlo', 'Monaco', 'Courier New', monospace
Font Sizes:
  - Navbar: 14px (0.875rem)
  - Panel Headers: 12px (0.75rem)
  - Status Bar: 11px (0.6875rem)
  - Terminal: 13px
```

#### Logo

```xml
<!-- ODEI Logo: Diamond shape with center dot -->
<svg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 32 33' fill='none'>
  <path d='M17.0714 28.5761...' fill='white'/>
  <path d='M16.0003 20.0282...' fill='white'/>
</svg>
```

### Component Specifications

#### 1. Navbar

**Location**: Top, full width
**Height**: 52px
**Background**: `#1f2937` (gray-900)

**Elements**:

- **Logo + Title**: Left-aligned, 24px logo + "ODEI" text
- **Module Tabs**: 7 tabs (Discuss, Plan, Execute, Mind, Body, Finance, Builder) with Memory as a dedicated view
  - Active state: `bg-gray-700`, full opacity
  - Inactive state: `bg-transparent`, 60% opacity, hover effect
  - Status indicator: 8px dot (green = running, red = stopped)
  - Icons: Lucide icons (message-square, scale, rocket, brain)

**Behavior**:

- Click tab → switches active module
- Active module has xterm terminal visible
- Smooth transition (200ms ease)

#### 2. Terminal View

**Location**: Left pane (flex-grow)
**Background**: `#000000` (pure black)

**Implementation**:

```typescript
// xterm.js configuration
{
  cursorBlink: true,
  fontSize: 13,
  fontFamily: 'Menlo, Monaco, "Courier New", monospace',
  lineHeight: 1.4,
  convertEol: true,
  scrollback: 10000,
  theme: {
    background: '#000000',
    foreground: '#E3E4E8',
    cursor: '#5E6AD2',
    selectionBackground: '#5E6AD2',
    black: '#000000',
    red: '#EF4444',
    green: '#10B981',
    yellow: '#F59E0B',
    blue: '#3B82F6',
    magenta: '#A855F7',
    cyan: '#06B6D4',
    white: '#E5E7EB',
  }
}
```

**Addons**:

- `FitAddon` — Auto-resize terminal to container
- `WebLinksAddon` — Clickable URLs (opens in external browser)
- `SearchAddon` — (Optional) Ctrl+F search within terminal

**PTY Integration**:

```javascript
// Each terminal connected to a PTY process
pty.spawn('claude', ['--interactive', '--dangerously-skip-permissions'], {
  cwd: `/Users/ai/ODEI/agents/${agentName}`,
  env: process.env,
  cols: 120,
  rows: 32,
  name: 'xterm-color',
});
```

**Output Handling**:

- Throttled to 16ms minimum (prevent UI freeze)
- Buffered in chunks (max 4KB per write)
- Scrollback limit: 10,000 lines (auto-trim older)

#### 3. Resizable Divider

**Location**: Between terminal and data panel
**Width**: 4px (1px visible + 3px hit area)
**Color**: `#374151` (gray-700), hover: `#5E6AD2` (indigo-500)

**Behavior**:

- Drag to resize data panel width
- Min width: 280px
- Max width: 800px
- Default width: 400px
- Cursor changes to `col-resize` on hover
- Terminal auto-refits after resize

#### 4. Data Panel

**Location**: Right pane
**Width**: Resizable (default 400px)
**Background**: `#1f2937` (gray-900)

**Sections**:

##### A. Agent Status

```
┌─────────────────────────┐
│ 🔄 Agent Status         │
├─────────────────────────┤
│ Status:     Running     │
│ Retries:    0           │
│ Last Activity: 18:30:45 │
└─────────────────────────┘
```

**Data Source**: `agent-manager.js` state

##### B. Control Frames (Last 5)

```
┌─────────────────────────┐
│ 🚨 Control Frames       │
├─────────────────────────┤
│ READY    18:30:45       │
│ {"stage":"discuss",...} │
├─────────────────────────┤
│ HANDOFF  18:31:12       │
│ decisions               │
└─────────────────────────┘
```

**Data Source**: Parsed from PTY output
**Frame Types**: READY, HANDOFF, ERROR, THOUGHT

##### C. Recent Events (Optional)

- MCP tool calls
- Decision recordings
- Task creations
- Pattern discoveries

**Data Source**: MCP server logs (future)

#### 5. Status Bar

**Location**: Bottom, full width
**Height**: 32px
**Background**: `#1f2937` (gray-900)

**Elements**:

- **Left Side**: Module name + Agent status
  - `Module: discuss • Agent: Running`
- **Right Side**: Health indicators
  - Neo4j: 🗄️ ● (green/red dot)
  - Apple Calendar: 📅 ●
  - Apple Notes: 📝 ●
  - History: 📊 ●

**Health Check**:

```javascript
// Every 5 seconds
{
  neo4j: true/false,
  mcp: {
    'odei-neo4j': { status: 'ok' | 'error', latency: 45 },
    'odei-history': { status: 'ok' | 'error', latency: 12 },
    'odei-apple': { status: 'ok' | 'error', latency: 8 }
  }
}
```

---

## 🔧 Technical Architecture

### File Structure

```
odei-electron/
├── package.json
├── electron/
│   ├── main.js                 # Electron main process
│   ├── preload.js              # Context bridge (IPC)
│   ├── agent-manager.js        # PTY spawning & control frame parsing
│   ├── health-probes.js        # MCP server health monitoring
│   └── ipc-handlers.js         # IPC method implementations
├── src/
│   ├── index.html              # Entry point
│   ├── renderer.js             # UI logic
│   ├── components/
│   │   ├── Navbar.js
│   │   ├── TerminalView.js
│   │   ├── DataPanel.js
│   │   └── StatusBar.js
│   └── styles/
│       └── main.css            # Tailwind + custom styles
├── assets/
│   ├── logo.svg
│   └── icon.icns / icon.ico
└── .env
```

### Dependencies

```json
{
  "name": "odei-electron",
  "version": "1.0.0",
  "main": "electron/main.js",
  "scripts": {
    "start": "electron .",
    "dev": "electron . --enable-logging",
    "build:mac": "electron-builder --mac",
    "build:win": "electron-builder --win",
    "build:linux": "electron-builder --linux"
  },
  "dependencies": {
    "electron": "^28.0.0",
    "node-pty": "^1.0.0",
    "xterm": "^5.3.0",
    "xterm-addon-fit": "^0.8.0",
    "xterm-addon-web-links": "^0.9.0",
    "dotenv": "^16.3.1",
    "zod": "^3.22.4"
  },
  "devDependencies": {
    "electron-builder": "^24.9.1"
  }
}
```

### Main Process (electron/main.js)

**Responsibilities**:

1. Create BrowserWindow with security settings
2. Load preload script
3. Initialize AgentManager
4. Initialize HealthProbes
5. Handle IPC from renderer
6. Manage app lifecycle

**Security Configuration**:

```javascript
const mainWindow = new BrowserWindow({
  width: 1600,
  height: 900,
  minWidth: 1200,
  minHeight: 700,
  backgroundColor: '#0a0a0f',
  webPreferences: {
    preload: path.join(__dirname, 'preload.js'),
    contextIsolation: true, // ✅ Isolate renderer
    nodeIntegration: false, // ✅ No Node.js in renderer
    sandbox: true, // ✅ Sandbox renderer
    webSecurity: true,
    allowRunningInsecureContent: false,
  },
});

// CSP Header
mainWindow.webContents.session.webRequest.onHeadersReceived((details, callback) => {
  callback({
    responseHeaders: {
      ...details.responseHeaders,
      'Content-Security-Policy': [
        "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' data:;",
      ],
    },
  });
});
```

**Window State**:

- Remember window position/size
- Restore on relaunch
- Support multiple monitors

### Preload Script (electron/preload.js)

**Purpose**: Expose safe IPC methods to renderer

```javascript
const { contextBridge, ipcRenderer } = require('electron');

contextBridge.exposeInMainWorld('electronAPI', {
  // Agent management
  getAgentOutput: (agent) => ipcRenderer.invoke('agent:get-output', agent),
  sendToAgent: (agent, data) => ipcRenderer.send('agent:send-input', agent, data),
  resizeAgent: (agent, cols, rows) => ipcRenderer.send('agent:resize', agent, cols, rows),
  isAgentRunning: (agent) => ipcRenderer.invoke('agent:is-running', agent),

  // Health
  getHealth: () => ipcRenderer.invoke('health:get'),

  // Event listeners
  onAgentOutput: (callback) => {
    const listener = (event, data) => callback(data);
    ipcRenderer.on('agent:output', listener);
    return () => ipcRenderer.removeListener('agent:output', listener);
  },

  onAgentError: (callback) => {
    const listener = (event, data) => callback(data);
    ipcRenderer.on('agent:error', listener);
    return () => ipcRenderer.removeListener('agent:error', listener);
  },

  onHealthUpdate: (callback) => {
    const listener = (event, data) => callback(data);
    ipcRenderer.on('health:update', listener);
    return () => ipcRenderer.removeListener('health:update', listener);
  },

  onAgentEvent: (callback) => {
    const listener = (event, data) => callback(data);
    ipcRenderer.on('agent:event', listener);
    return () => ipcRenderer.removeListener('agent:event', listener);
  },

  // External links
  openExternal: (url) => ipcRenderer.send('open-external', url),
});
```

**Validation**:

- All IPC methods use Zod schemas
- No arbitrary method execution
- Whitelist of allowed operations

### Agent Manager (electron/agent-manager.js)

**Core Class**:

```javascript
const pty = require('node-pty');
const EventEmitter = require('events');

class AgentManager extends EventEmitter {
  constructor(agentsDir, env) {
    super();
    this.agentsDir = agentsDir;
    this.env = env;
    this.agents = new Map(); // agentName -> { pty, buffer, state }
    this.maxRetries = 5;
    this.baseBackoffMs = 1000;
  }

  startAgent(agentName) {
    const agentDir = path.join(this.agentsDir, agentName);
    const claudeBin = this.env.CLAUDE_BIN || 'claude';
    const claudeFlags = (this.env.CLAUDE_FLAGS || '--interactive --dangerously-skip-permissions').split(' ');

    const ptyProcess = pty.spawn(claudeBin, claudeFlags, {
      name: 'xterm-color',
      cols: 120,
      rows: 32,
      cwd: agentDir,
      env: { ...process.env, TERM: 'xterm-256color', FORCE_COLOR: '1' },
    });

    this.agents.set(agentName, {
      pty: ptyProcess,
      buffer: [],
      state: { running: true, retries: 0, lastActivity: Date.now() },
    });

    // Output handling with throttling
    let outputBuffer = '';
    let throttleTimeout = null;

    ptyProcess.onData((data) => {
      outputBuffer += data;

      if (!throttleTimeout) {
        throttleTimeout = setTimeout(() => {
          this.emit('output', { agent: agentName, data: outputBuffer });
          this._scanControlFrames(agentName, outputBuffer);
          outputBuffer = '';
          throttleTimeout = null;
        }, 16); // 60fps max
      }

      // Update last activity
      this.agents.get(agentName).state.lastActivity = Date.now();
    });

    // Exit handling with exponential backoff
    ptyProcess.onExit(({ exitCode, signal }) => {
      const agent = this.agents.get(agentName);
      agent.state.running = false;

      if (agent.state.retries < this.maxRetries) {
        const backoffMs = this.baseBackoffMs * Math.pow(2, agent.state.retries);
        agent.state.retries++;

        this.emit('error', {
          agent: agentName,
          error: `Process exited (code: ${exitCode}, signal: ${signal})`,
          retryIn: backoffMs,
        });

        setTimeout(() => this.startAgent(agentName), backoffMs);
      } else {
        this.emit('error', {
          agent: agentName,
          error: 'Max retries exceeded',
          fatal: true,
        });
      }
    });
  }

  _scanControlFrames(agentName, data) {
    const frameRegex = /<<<ODEI:([A-Z_]+):(.*?)>>>/g;
    let match;

    while ((match = frameRegex.exec(data)) !== null) {
      this.emit('control-frame', {
        agent: agentName,
        type: match[1],
        payload: match[2],
        timestamp: Date.now(),
      });

      // Special handling for HANDOFF
      if (match[1] === 'HANDOFF') {
        this.emit('handoff', {
          from: agentName,
          to: match[2],
          timestamp: Date.now(),
        });
      }
    }
  }

  writeToAgent(agentName, data) {
    const agent = this.agents.get(agentName);
    if (agent?.pty) {
      agent.pty.write(data);
    }
  }

  resizeAgent(agentName, cols, rows) {
    const agent = this.agents.get(agentName);
    if (agent?.pty) {
      agent.pty.resize(cols, rows);
    }
  }

  killAgent(agentName) {
    const agent = this.agents.get(agentName);
    if (agent?.pty) {
      agent.pty.kill();
      this.agents.delete(agentName);
    }
  }

  killAll() {
    for (const [name, agent] of this.agents) {
      agent.pty.kill();
    }
    this.agents.clear();
  }

  getAgentState(agentName) {
    return this.agents.get(agentName)?.state || null;
  }
}

module.exports = AgentManager;
```

### Health Probes (electron/health-probes.js)

**Purpose**: Monitor MCP server health

```javascript
const { spawn } = require('child_process');

class HealthProbes {
  constructor(mcpConfig) {
    this.mcpConfig = mcpConfig;
    this.healthCache = {};
    this.checkInterval = 5000; // 5 seconds
  }

  async checkNeo4j() {
    const uri = process.env.NEO4J_URI || 'bolt://localhost:7687';
    try {
      // Simple TCP connection check
      const { default: net } = await import('net');
      return await new Promise((resolve) => {
        const client = net.connect({ host: 'localhost', port: 7687 });
        client.on('connect', () => {
          client.end();
          resolve(true);
        });
        client.on('error', () => resolve(false));
        setTimeout(() => {
          client.end();
          resolve(false);
        }, 1000);
      });
    } catch {
      return false;
    }
  }

  async checkMCPServer(serverName) {
    const serverConfig = this.mcpConfig.mcpServers[serverName];
    if (!serverConfig) return { status: 'unknown' };

    const startTime = Date.now();

    try {
      const proc = spawn(serverConfig.command, serverConfig.args, {
        env: { ...process.env, ...serverConfig.env },
        stdio: ['pipe', 'pipe', 'pipe'],
      });

      const healthCheck = new Promise((resolve, reject) => {
        proc.stdin.write('{"jsonrpc":"2.0","id":1,"method":"health.ping"}\n');

        let output = '';
        proc.stdout.on('data', (data) => {
          output += data.toString();
          if (output.includes('"result":"pong"')) {
            proc.kill();
            resolve({ status: 'ok', latency: Date.now() - startTime });
          }
        });

        setTimeout(() => {
          proc.kill();
          reject(new Error('Timeout'));
        }, 2000);
      });

      return await healthCheck;
    } catch (error) {
      return { status: 'error', error: error.message };
    }
  }

  async checkAll() {
    const [neo4j, ...mcpResults] = await Promise.all([
      this.checkNeo4j(),
      this.checkMCPServer('odei-neo4j'),
      this.checkMCPServer('odei-history'),
      this.checkMCPServer('odei-apple'),
    ]);

    return {
      neo4j,
      mcp: {
        'odei-neo4j': mcpResults[0],
        'odei-history': mcpResults[1],
        'odei-apple': mcpResults[2],
      },
      timestamp: Date.now(),
    };
  }

  startMonitoring(callback) {
    this.intervalId = setInterval(async () => {
      const health = await this.checkAll();
      this.healthCache = health;
      callback(health);
    }, this.checkInterval);

    // Initial check
    this.checkAll().then(callback);
  }

  stopMonitoring() {
    if (this.intervalId) {
      clearInterval(this.intervalId);
    }
  }
}

module.exports = HealthProbes;
```

---

## 🎬 User Workflows

### Workflow 1: Initial Launch

```
1. User opens ODEI app
   ↓
2. Electron main process starts
   ↓
3. AgentManager spawns 4 PTY processes
   - discuss
   - decisions
   - execute
   - mind
   ↓
4. Each agent emits: <<<ODEI:READY:{"stage":"..."}>>>
   ↓
5. UI shows all 4 tabs with green status dots
   ↓
6. Default tab: "Discuss" is active
   ↓
7. HealthProbes checks all MCP servers
   ↓
8. Status bar shows health: ✅✅✅
   ↓
9. User sees: "ODEI › Discuss Agent ready"
```

**Success Criteria**:

- All 4 terminals show READY within 5 seconds
- Health status all green within 10 seconds
- Memory usage < 350MB

### Workflow 2: Constitutional Decision

```
User in Discuss tab:
1. User types: "Should I take the corporate job offer?"
   ↓
2. Discuss agent:
   - Recalls past discussions (MCP: memory.recall)
   - Checks constitution (reads prompt.md context)
   - Performs Socratic questioning
   - Validates goal alignment (MCP: goals.hierarchy)
   - Outputs recommendation: APPROVE
   ↓
3. Discuss agent emits: <<<ODEI:HANDOFF:decisions>>>
   ↓
4. Electron app catches control frame
   ↓
5. UI automatically switches to "Decisions" tab
   ↓
6. Decisions agent:
   - Reads discussion thread (MCP: history.threads.get)
   - Calculates ROI for each option
   - Shows breakdown to user
   - User approves top choice
   ↓
7. Decisions agent records in Neo4j (MCP: decisions.create)
   ↓
8. Emits: <<<ODEI:HANDOFF:execute>>>
   ↓
9. UI switches to "Execute" tab
   ↓
10. Execute agent:
    - Fetches decision (MCP: decisions.pending)
    - Decomposes into tasks
    - Analyzes calendar (MCP: calendar.window)
    - Proposes time blocks
    - User approves
    ↓
11. Execute agent creates events (MCP: calendar.create with confirm=true)
    ↓
12. Emits: <<<ODEI:HANDOFF:mind>>>
    ↓
13. Mind agent runs background analysis
```

**Success Criteria**:

- Handoffs happen automatically (no manual tab switching)
- Data panel shows control frames in real-time
- No data loss between stages
- All MCP calls succeed

### Workflow 3: Manual Agent Interaction

```
1. User clicks "Execute" tab
   ↓
2. UI shows Execute terminal
   ↓
3. User types command in terminal
   ↓
4. Command sent to PTY process
   ↓
5. Execute agent processes command
   ↓
6. Output appears in terminal
   ↓
7. Data panel updates with agent status
```

**Success Criteria**:

- Terminal input is responsive (<50ms)
- Output appears smoothly (throttled, no freeze)
- Scrollback works correctly

### Workflow 4: Agent Crash Recovery

```
1. Agent process crashes (e.g., Execute)
   ↓
2. PTY emits exit event
   ↓
3. AgentManager catches exit
   ↓
4. Status dot turns red
   ↓
5. Data panel shows: "Agent stopped, retrying in 1s..."
   ↓
6. AgentManager waits (exponential backoff)
   ↓
7. Restarts agent (retry 1)
   ↓
8. If successful: Status dot turns green
   ↓
9. Terminal shows reconnection message
```

**Success Criteria**:

- No data loss from terminal buffer
- Max 5 retries before giving up
- User is informed of retry attempts
- After max retries, shows manual restart button

---

## 📊 Performance Requirements

### Memory Usage

- **Target**: <350MB with 4 terminals idle
- **Max**: <600MB with 4 terminals active + heavy output
- **Method**: node-pty uses ~50MB per PTY, xterm.js ~30MB per instance

### Startup Time

- **Target**: <2.5 seconds (cold start)
- **Max**: <5 seconds on older hardware
- **Breakdown**:
  - Electron init: 500ms
  - Window creation: 200ms
  - Agent spawning (parallel): 1000ms
  - xterm.js init: 500ms
  - Health checks: 300ms

### Terminal Responsiveness

- **Input latency**: <50ms (keystroke to PTY write)
- **Output latency**: <100ms (PTY data to xterm display)
- **Throttling**: 16ms minimum (60fps), max 4KB per chunk

### Resize Performance

- **Data panel resize**: 60fps smooth drag
- **Terminal refit**: <100ms after resize complete
- **Window resize**: All terminals refit in <200ms

---

## 🔐 Security Requirements

### Renderer Isolation

- ✅ `contextIsolation: true`
- ✅ `nodeIntegration: false`
- ✅ `sandbox: true`
- ✅ CSP enforced

### IPC Validation

- ✅ All IPC methods have Zod schemas
- ✅ Whitelist of allowed operations
- ✅ No arbitrary code execution

### MCP Write Operations

- ✅ Default `dryRun: true`
- ✅ Calendar writes require `confirm: true`
- ✅ User must explicitly approve

### Secrets Management

- ✅ `.env` never committed
- ✅ NEO4J_PASSWORD not logged
- ✅ MCP credentials from environment only

### External Links

- ✅ `openExternal` for all URLs
- ✅ Never navigate main window
- ✅ Whitelist: github.com, anthropic.com

---

## 🧪 Testing Requirements

### Unit Tests

- AgentManager PTY spawning
- Control frame parsing
- Exponential backoff logic
- Health probe checks

### Integration Tests

- All 4 agents start successfully
- Control frames trigger handoffs
- MCP servers respond to health checks
- Terminal input/output flow

### E2E Tests (Playwright)

```javascript
test('Full workflow: Discuss → Decisions → Execute', async () => {
  // 1. Launch app
  await app.launch();

  // 2. Verify all agents ready
  await expect(page.locator('[data-agent="discuss"] .status-dot')).toHaveClass(/bg-green/);

  // 3. Type in Discuss terminal
  await page.locator('[data-module="discuss"] .xterm').type('Should I launch ODEI?\n');

  // 4. Wait for handoff
  await expect(page.locator('[data-module="decisions"]')).toHaveClass(/active/);

  // 5. Verify control frame appeared
  await expect(page.locator('.control-frame')).toContainText('HANDOFF');
});
```

### Performance Tests

- Startup time <2.5s (100 runs average)
- Memory usage <350MB idle
- Terminal output handles 10,000 lines/sec

---

## 🚀 Build & Distribution

### Electron Builder Config

```json
{
  "appId": "com.odei.app",
  "productName": "ODEI",
  "directories": {
    "output": "dist"
  },
  "files": ["electron/**/*", "src/**/*", "assets/**/*", "node_modules/**/*"],
  "asarUnpack": ["node_modules/node-pty/**/*"],
  "mac": {
    "category": "public.app-category.productivity",
    "icon": "assets/icon.icns",
    "target": ["dmg", "zip"],
    "darkModeSupport": true
  },
  "win": {
    "target": ["nsis", "portable"],
    "icon": "assets/icon.ico"
  },
  "linux": {
    "target": ["AppImage", "deb"],
    "category": "Office"
  }
}
```

**Platform Notes**:

- **macOS**: Full support (Calendar integration works)
- **Windows**: Calendar tools return `UNSUPPORTED_PLATFORM`
- **Linux**: Calendar tools return `UNSUPPORTED_PLATFORM`

### Auto-Updates (Future)

- `electron-updater` for delta updates
- Check for updates on launch
- Background download
- User prompt before install

---

## 📋 Definition of Done

### Phase 2 Complete When:

- [ ] All 4 PTY terminals spawn successfully
- [ ] xterm.js displays output correctly
- [ ] Control frames parsed and handled
- [ ] Agent handoffs trigger UI tab switches
- [ ] Health status panel shows all MCP servers
- [ ] Data panel shows agent status + control frames
- [ ] Resizable panel works smoothly
- [ ] Terminal input is responsive (<50ms)
- [ ] Memory usage <350MB idle
- [ ] Startup time <2.5s
- [ ] Sandbox + CSP enforced
- [ ] No direct Neo4j/SQLite access from renderer
- [ ] Builds successfully: .dmg (macOS), .exe (Windows), .AppImage (Linux)
- [ ] All E2E tests pass

---

## 🎯 Next Steps (Implementation Order)

1. **Setup Project** (30 min)
   - Create package.json
   - Install dependencies
   - Setup folder structure

2. **Build Main Process** (2 hours)
   - electron/main.js with security
   - electron/preload.js with IPC bridge
   - Window state management

3. **Build AgentManager** (3 hours)
   - PTY spawning logic
   - Control frame parsing
   - Exponential backoff
   - Event emitters

4. **Build HealthProbes** (1 hour)
   - MCP server health checks
   - Neo4j connection test
   - Periodic monitoring

5. **Build UI** (4 hours)
   - HTML structure (based on prototype)
   - xterm.js integration
   - Data panel rendering
   - Status bar

6. **IPC Wiring** (2 hours)
   - Connect UI to AgentManager
   - Connect UI to HealthProbes
   - Event listeners

7. **Testing** (2 hours)
   - Manual testing all workflows
   - Fix bugs
   - Performance optimization

8. **Build & Package** (1 hour)
   - electron-builder config
   - Test .dmg build
   - Create installer

**Total Estimate**: ~15 hours (2 work days)

---

**Ready to build when you give the signal.** 🚀
