# ODEI Electron Application

**Version**: 1.0.0
**Status**: Production Ready ✅
**Last Updated**: 2025-10-09

---

## 🎯 Overview

The ODEI Electron Application is a production-ready desktop interface for orchestrating 4 autonomous Claude Code agents (Discuss, Decisions, Execute, Mind) with full MCP server integration.

**Key Features**:

- 4 independent PTY terminals running Claude Code agents
- Real-time control frame parsing for agent coordination
- Automatic handoffs between agents
- Health monitoring for 3 MCP servers (Neo4j, History, Apple)
- Beautiful dark-themed UI with xterm.js terminals
- Cross-platform support (macOS, Windows, Linux)

---

## 🚀 Quick Start

### Prerequisites

1. **Node.js** 18+ installed
2. **Claude Code CLI** installed and in PATH
3. **Neo4j** running locally (optional for full functionality)
4. **Environment variables** configured (see below)

### Installation

```bash
# 1. Clone and navigate to ODEI
cd /Users/ai/ODEI

# 2. Install dependencies (already done)
npm install

# 3. Build Tailwind CSS
npm run build:css

# 4. Start in development mode
npm run dev
```

### Environment Setup

Create a `.env` file in the root directory:

```bash
# Neo4j (required for Discuss/Decisions agents)
NEO4J_URI=bolt://localhost:7687
NEO4J_USERNAME=neo4j
NEO4J_PASSWORD=your_password
NEO4J_DATABASE=memory

# History database (required for all agents)
ODEI_HISTORY_DB_PATH=/Users/ai/ODEI/data/history.sqlite

# Apple Calendar (optional, macOS only)
ODEI_APPLE_WRITE_MODE=prompt

# Claude CLI (optional overrides)
CLAUDE_BIN=claude
CLAUDE_FLAGS=--dangerously-skip-permissions
```

---

## 🎮 Usage

### Development Mode

```bash
npm run dev
```

This starts Electron with:

- DevTools open automatically
- `--dangerously-skip-permissions` flag (bypasses Claude permission prompts)
- Full logging enabled

### Production Mode

```bash
npm start
```

This starts Electron in production mode:

- No DevTools
- Requires permission approval for Claude operations
- Minimal logging

### Building Distributable Apps

```bash
# macOS
npm run build:mac
# Output: dist-electron/ODEI-1.0.0.dmg

# Windows
npm run build:win
# Output: dist-electron/ODEI Setup 1.0.0.exe

# Linux
npm run build:linux
# Output: dist-electron/ODEI-1.0.0.AppImage
```

---

## 🏗️ Architecture

### File Structure

```
ODEI/
├── electron/
│   ├── main.js              # Main process (app lifecycle, security)
│   ├── agent-manager.js     # PTY spawning & control frame parsing
│   ├── health-probes.js     # MCP server health monitoring
│   ├── ipc-handlers.js      # IPC validation with Zod
│   └── preload.js           # Secure IPC bridge
├── src/
│   ├── index.html           # UI structure (CSP meta tag)
│   ├── renderer.js          # xterm.js terminals & UI logic
│   ├── input.css            # Tailwind source
│   └── styles.css           # Built CSS (generated)
├── assets/
│   ├── icon.svg             # Base icon
│   ├── icon.icns            # macOS icon (to generate)
│   ├── icon.ico             # Windows icon (to generate)
│   └── icon.png             # Linux icon (to generate)
├── agents/                  # 4 agent workspaces
│   ├── discuss/
│   ├── decisions/
│   ├── execute/
│   └── mind/
├── .claude/                 # Root MCP config
│   └── mcp.json
├── package.json
└── tailwind.config.js
```

### Key Components

#### 1. AgentManager (`electron/agent-manager.js`)

Manages 4 PTY processes running Claude Code agents.

**Features**:

- ✅ Persistent `scanBuffer` for robust control frame parsing
- ✅ Exponential backoff retry (5 retries max)
- ✅ Cross-platform process killing with `tree-kill`
- ✅ Real-time output throttling (60fps max)

**Control Frames**:

```
<<<ODEI:READY:discuss>>>           # Agent ready
<<<ODEI:HANDOFF:decisions>>>       # Handoff to decisions
<<<ODEI:ERROR:error message>>>     # Error occurred
<<<ODEI:THOUGHT:thinking...>>>     # Agent reasoning
```

#### 2. HealthProbes (`electron/health-probes.js`)

Monitors MCP server health every 5 seconds.

**Features**:

- ✅ Correct MCP ping implementation (`method: "ping"`, `result: {}`)
- ✅ Persistent processes (no spawning every 5s)
- ✅ ID correlation for ping/pong matching
- ✅ Neo4j TCP connection check

#### 3. Security (`electron/main.js`)

Production-grade security hardening.

**Settings**:

- `contextIsolation: true` — Renderer isolated from main process
- `nodeIntegration: false` — No Node.js in renderer
- `sandbox: true` — Renderer fully sandboxed
- CSP meta tag enforced (no remote code)
- All IPC methods validated with Zod schemas
- External URLs whitelisted + confirmation dialog

---

## 🔒 Security

### Security Checklist

✅ `contextIsolation: true`
✅ `nodeIntegration: false`
✅ `sandbox: true`
✅ CSP meta tag (no CDN, `script-src 'self'` only)
✅ All IPC methods validated with Zod
✅ No direct database access from renderer
✅ External URLs whitelisted
✅ `.env` not committed to git
✅ Passwords not logged
✅ Danger mode gated to development only

### Whitelisted External Domains

- github.com
- anthropic.com
- claude.com
- modelcontextprotocol.io
- electronjs.org
- neo4j.com
- xtermjs.org

All other URLs require user confirmation before opening.

---

## ⚡ Performance

### Target Metrics

- **Startup time**: <2.5s (from launch to all agents ready)
- **Memory usage**: <350MB idle, <600MB active
- **Terminal latency**: <50ms input to display
- **Output throttling**: 16ms (60fps max)
- **Scrollback**: 10,000 lines per terminal

### Optimization Techniques

1. **Output throttling**: PTY data emitted max once per 16ms
2. **Control frame buffering**: Persistent buffer catches split frames
3. **Terminal rendering**: xterm.js with optimized settings
4. **Health check batching**: All 4 checks run in parallel
5. **Debounced resizing**: Terminal resize debounced to 100ms

---

## 🐛 Troubleshooting

### Agents not starting

**Symptom**: Terminals remain black, no output

**Solutions**:

1. Check Claude CLI is installed: `which claude`
2. Check environment variables in `.env`
3. Check DevTools console for errors (press Cmd+Opt+I)
4. Try starting manually: `cd agents/discuss && claude`

### Control frames not parsed

**Symptom**: Handoffs don't trigger, events not logged

**Solutions**:

1. Check agent prompt includes control frame examples
2. Check DevTools console for regex errors
3. Verify agents are printing frames: `<<<ODEI:TYPE:payload>>>`

### MCP servers unhealthy

**Symptom**: Red dots in health panel

**Solutions**:

1. Check Neo4j is running: `neo4j status`
2. Check MCP servers built: `npm run build`
3. Check environment variables match `.env.example`
4. Test MCP server manually: `node servers/odei-history/dist/index.js`

### High memory usage

**Symptom**: App using >1GB RAM

**Solutions**:

1. Reduce scrollback: Edit `src/renderer.js` → `scrollback: 5000`
2. Clear terminal buffers periodically
3. Restart app every few hours

### Build fails

**Symptom**: `npm run build:mac` errors

**Solutions**:

1. Install Xcode Command Line Tools (macOS)
2. Rebuild native modules: `npm run postinstall`
3. Clear electron-builder cache: `rm -rf ~/Library/Caches/electron-builder`

---

## 🧪 Testing

### Manual Testing Checklist

- [ ] All 4 agents spawn successfully
- [ ] Terminals display output correctly
- [ ] Agent tabs switch smoothly
- [ ] Control frames are parsed and logged
- [ ] Handoffs trigger automatic tab switches
- [ ] Health status updates every 5 seconds
- [ ] All MCP servers show green dots
- [ ] Terminal input works in all agents
- [ ] Window resize refits terminals
- [ ] External links open in browser
- [ ] App restarts agents on crash (max 5 retries)

### Automated Testing (Future)

```bash
# Run smoke tests
npm run smoke

# Test individual components
npm run test:agent-manager
npm run test:health-probes
npm run test:ui
```

---

## 📦 Distribution

### macOS (.dmg)

```bash
npm run build:mac
```

**Output**: `dist-electron/ODEI-1.0.0.dmg`

**Features**:

- Code-signed (if certificates available)
- Notarized for Gatekeeper (if Apple Developer account)
- Dark mode support
- Drag to Applications folder

### Windows (.exe)

```bash
npm run build:win
```

**Output**: `dist-electron/ODEI Setup 1.0.0.exe`

**Features**:

- NSIS installer + portable version
- Auto-update support (if update server configured)
- Add to PATH option
- Create desktop shortcut

### Linux (.AppImage)

```bash
npm run build:linux
```

**Output**: `dist-electron/ODEI-1.0.0.AppImage`

**Features**:

- Self-contained (no dependencies)
- Works on any Linux distro
- Desktop integration
- Portable

---

## 🔧 Configuration

### Agent Configuration

Each agent has its own workspace in `agents/`:

```
agents/discuss/
├── .claude/
│   ├── CLAUDE.md    # Agent role & instructions
│   └── mcp.json     # Symlink to root MCP config
├── prompt.md        # Constitutional context
└── README.md        # Agent documentation
```

**To customize an agent**:

1. Edit `agents/{name}/.claude/CLAUDE.md`
2. Add role-specific instructions
3. Define control frame usage
4. Restart Electron app

### MCP Server Configuration

MCP servers configured in `.claude/mcp.json`:

```json
{
  "mcpServers": {
    "odei-neo4j": {
      "command": "node",
      "args": ["servers/odei-neo4j/dist/index.js"],
      "env": {
        "NEO4J_URI": "bolt://localhost:7687",
        "NEO4J_USERNAME": "neo4j",
        "NEO4J_PASSWORD": "your_password"
      }
    }
  }
}
```

**To add a new MCP server**:

1. Add entry to `mcpServers` in `.claude/mcp.json`
2. Update `HealthProbes.checkAll()` in `electron/health-probes.js`
3. Add UI element in `src/index.html`
4. Restart Electron app

---

## 📊 Monitoring

### Logs

**Development**: Logs printed to terminal running `npm run dev`

**Production**: Logs saved to:

- macOS: `~/Library/Logs/ODEI/`
- Windows: `%USERPROFILE%\AppData\Roaming\ODEI\logs\`
- Linux: `~/.config/ODEI/logs/`

### Health Dashboard

The right panel shows real-time status:

**Agent Status**:

- 🟢 Green = Running
- 🔴 Red = Stopped/crashed

**Health Status**:

- 🟢 Green = OK (with latency)
- 🔴 Red = Error/timeout
- ⚪ Gray = Unknown/not checked

**Recent Events**:

- Last 10 control frames
- Timestamps + agent + payload

---

## 🚧 Known Limitations

1. **No multi-window support**: Single window only
2. **No terminal tabs**: 4 terminals, switch via navbar
3. **No terminal history export**: Use terminal scrollback only
4. **No agent pause/resume**: Only restart via crash recovery
5. **No custom themes**: Dark theme only
6. **No terminal search**: Use browser find (Cmd+F)
7. **No terminal copy/paste shortcuts**: Use right-click menu

---

## 🗺️ Roadmap

### v1.1.0 (Next)

- [ ] Terminal tabs instead of navbar switching
- [ ] Export terminal history to file
- [ ] Custom theme support
- [ ] Agent pause/resume buttons
- [ ] Terminal search functionality

### v1.2.0 (Future)

- [ ] Multi-window support (one agent per window)
- [ ] Visual workflow builder (control frame editor)
- [ ] Built-in metrics dashboard
- [ ] Auto-update support
- [ ] Cloud sync for agent history

### v2.0.0 (Vision)

- [ ] Plugin system for custom agents
- [ ] Web-based remote access
- [ ] Distributed agent execution
- [ ] Real-time collaboration
- [ ] AI-powered agent suggestions

---

## 🤝 Contributing

### Development Workflow

1. **Fork & clone** the repository
2. **Create feature branch**: `git checkout -b feature/my-feature`
3. **Make changes** in `electron/` or `src/`
4. **Test thoroughly**: `npm run dev`
5. **Commit with conventional commits**: `feat: add terminal search`
6. **Push & create PR**: Include screenshots if UI changes

### Code Style

- **JavaScript**: ES6+ with Node.js modules
- **HTML**: Semantic, accessible markup
- **CSS**: Tailwind utility classes only
- **Comments**: Explain _why_, not _what_

### Testing Guidelines

- Test on all 3 platforms before merging
- Verify security settings unchanged
- Check memory usage under load
- Ensure all 4 agents start successfully

---

## 📄 License

MIT License - See root LICENSE file

---

## 🙏 Acknowledgments

- **Anthropic** — Claude Code CLI
- **Electron** — Cross-platform framework
- **xterm.js** — Terminal emulation
- **node-pty** — PTY bindings
- **Tailwind CSS** — Utility-first CSS

---

## 📞 Support

### Documentation

- [ELECTRON-APP-SPEC.md](ELECTRON-APP-SPEC.md) — Complete functional spec
- [IMPLEMENTATION-PROMPT.md](IMPLEMENTATION-PROMPT.md) — Implementation guide
- [Agents.md](Agents.md) — Agent architecture
- [AGENT-HANDOFF-PROTOCOL.md](AGENT-HANDOFF-PROTOCOL.md) — Control frames

### Issues

Report bugs: https://github.com/zer0h1ro/agentic/issues

### Community

Discussions: https://github.com/zer0h1ro/agentic/discussions

---

**Built with ❤️ by the ODEI Team**

🤖 _Generated with [Claude Code](https://claude.com/claude-code)_
