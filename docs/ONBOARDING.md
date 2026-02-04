# ODEI Developer Onboarding Guide

Welcome to ODEI — an Agentic Operating System for AI-assisted decision-making and knowledge management.

**Goal:** Get you productive in 30 minutes.

---

## What is ODEI?

ODEI is a cross-platform Electron application that runs multiple AI agents to help with:

- Constitutional decision-making (Discuss)
- Strategic planning (Plan)
- Task execution (Execute)
- Pattern analysis (Mind)
- Health intelligence (Health)
- Financial management (Finance)

All powered by a **7-layer graph architecture** in Neo4j, with conversation history in SQLite, and native integrations with Apple Calendar, Telegram, Miro, and health data sources.

---

## Prerequisites

Before starting, ensure you have:

| Tool | Version | Required For | Install |
|------|---------|--------------|---------|
| **Node.js** | 22+ | All development | [nodejs.org](https://nodejs.org) |
| **Neo4j Desktop** | 5.x | Graph database | [neo4j.com/download](https://neo4j.com/download) |
| **Git** | Latest | Version control | Pre-installed on macOS/Linux |
| **Swift** | 5.9+ | odei-apple (macOS only) | Pre-installed on macOS |
| **OpenAI API Key** | N/A | Embeddings | [platform.openai.com](https://platform.openai.com) |

**Optional:**
- Telegram Bot Token (for Telegram integration)
- Notion API Token (for Notion integration)
- Garmin credentials (for fitness data)

---

## Quick Start (30 Minutes)

### Step 1: Clone and Install (5 minutes)

```bash
# Clone repository
git clone <repository-url>
cd ODEI

# Install all dependencies
npm install
```

### Step 2: Set Up Neo4j (10 minutes)

1. **Install Neo4j Desktop** from [neo4j.com/download](https://neo4j.com/download)

2. **Create a new project** called "ODEI"

3. **Create a local database**:
   - Name: `ODEI Memory Atlas`
   - Password: Choose a strong password (you'll need this)
   - Version: 5.27+ recommended

4. **Install required plugins** (via Neo4j Desktop → Database → Plugins):
   - ✅ **Gen AI** (required for vector embeddings)
   - ⚠️ APOC (optional, not currently used)
   - ⚠️ Graph Data Science (optional, not currently used)

5. **Start the database** (green play button)

6. **Verify connection**:
   - Open Neo4j Browser (http://localhost:7474)
   - Run: `CALL db.ping()`
   - Should return success

### Step 3: Configure Environment (5 minutes)

```bash
# Copy environment template
cp .env.example .env

# Edit .env with your credentials
nano .env  # or use your preferred editor
```

**Required variables:**

```bash
# Neo4j
NEO4J_URI=bolt://127.0.0.1:7687
NEO4J_USERNAME=neo4j
NEO4J_PASSWORD=your_password_from_step2
NEO4J_DATABASE=memory

# OpenAI (for embeddings)
OPENAI_API_KEY=sk-proj-your_api_key_here
```

**Optional variables:**

```bash
# Telegram
TELEGRAM_BOT_TOKEN=your_bot_token_here
TELEGRAM_ALLOWED_CHAT_IDS=123456789

# Notion
NOTION_TOKEN=ntn_your_token_here
```

### Step 4: Build Everything (5 minutes)

```bash
# One-command setup (recommended for first time)
npm run setup
```

This automated script will:
- Build all MCP servers (TypeScript → JavaScript)
- Build Swift server (macOS only, odei-apple)
- Compile Tailwind CSS
- Create necessary directories

**Alternative: Manual build** (if automated setup fails):

```bash
# Build MCP servers
npm run build

# Build CSS
npm run build:css

# Build Swift server (macOS only)
cd servers/odei-apple && swift build -c release && cd ../..
```

### Step 5: Bootstrap Foundation Data (3 minutes)

```bash
# Populate Neo4j with core constitutional nodes
npm run odei:bootstrap
```

This creates:
- Foundation layer (Values, Principles, Guardrails)
- Vision layer (Goals, Visions)
- AI Principal and Human nodes
- ODEI Symbiosis partnership

**Verify bootstrap succeeded:**

```bash
# Open Neo4j Browser and run:
# MATCH (n) RETURN count(n)
# Should return 50+ nodes
```

### Step 6: Run ODEI (2 minutes)

```bash
# Development mode (with hot reload)
npm run dev
```

The application will launch in a new window. You should see:

- 5 agent tabs (Discuss, Plan, Execute, Mind, Finance)
- Health dashboard tab
- Memory Atlas visualization
- Terminal interface for each agent

**Alternative: Production mode:**

```bash
npm start
```

---

## Verify Your Setup

Run through this checklist to confirm everything works:

### 1. MCP Server Health Check

```bash
npm run smoke
```

Expected output:
```
✓ odei-history: OK
✓ odei-neo4j: OK
✓ odei-gemini: OK
✓ odei-telegram: OK
✓ odei-conductor: OK
✓ odei-health: OK
```

### 2. Neo4j Connection

Open Neo4j Browser and run:

```cypher
// Check foundation layer
MATCH (n)
WHERE n:Value OR n:Principle OR n:Guardrail
RETURN labels(n)[0] as type, count(n) as count
```

Should return counts for Value, Principle, Guardrail nodes.

### 3. Agent Terminals

In the ODEI app:

1. Click **Discuss** tab
2. Terminal should show: `discuss@odei:~$` prompt
3. Type: `echo "Hello ODEI"`
4. Should see the echo response

### 4. Memory Atlas Visualization

1. Click **Memory** tab
2. Should see graph visualization
3. Nodes should be arranged in a force-directed layout
4. Click any node to see details panel

### 5. Conversation Persistence

1. Open **Discuss** agent
2. Start conversation with Claude Code
3. Exit and restart ODEI
4. Conversation should persist (stored in SQLite)

---

## Common Issues & Solutions

### Issue: "Cannot connect to Neo4j"

**Symptoms:**
- MCP smoke test fails for odei-neo4j
- Error: `ServiceUnavailable: WebSocket connection failure`

**Solutions:**

1. Verify Neo4j is running:
   ```bash
   # Check if port 7687 is open
   nc -zv localhost 7687
   ```

2. Check credentials in `.env`:
   ```bash
   # Should match your Neo4j database password
   cat .env | grep NEO4J_PASSWORD
   ```

3. Verify database name:
   - Neo4j Desktop → Database → "memory" exists
   - Or change `NEO4J_DATABASE=neo4j` in `.env` to use default

4. Test connection directly:
   ```bash
   npm run start:neo4j
   # Should start without errors
   ```

### Issue: "Module did not self-register"

**Symptoms:**
- Electron fails to start
- Error: `Error: Module did not self-register`

**Solution:**

```bash
# Rebuild native modules for Electron
npm run rebuild:native
```

### Issue: "CSS not updating"

**Symptoms:**
- UI changes not reflected
- Tailwind classes not working

**Solution:**

```bash
# Rebuild CSS
npm run build:css

# Or run in watch mode during development
npm run dev:css
```

### Issue: "Permission denied" on scripts

**Symptoms:**
- `npm run setup` fails
- Error: `EACCES: permission denied`

**Solution:**

```bash
# Make scripts executable
chmod +x scripts/*.sh

# Retry setup
npm run setup
```

### Issue: "OpenAI API rate limit"

**Symptoms:**
- Bootstrap fails
- Error: `429 Too Many Requests`

**Solution:**

1. Wait 60 seconds and retry
2. Use environment variable to reduce batch size:
   ```bash
   BATCH_SIZE=5 npm run odei:bootstrap
   ```

### Issue: "Swift server won't build" (macOS)

**Symptoms:**
- Setup fails on Swift compilation
- odei-apple not available

**Solution:**

1. Verify Xcode Command Line Tools:
   ```bash
   xcode-select --install
   ```

2. Build manually with verbose output:
   ```bash
   cd servers/odei-apple
   swift build -c release -v
   ```

3. Skip Swift server (optional component):
   - Comment out Swift build in `scripts/dev-setup.sh`
   - ODEI will work without Apple Calendar integration

---

## Next Steps

Now that ODEI is running, here's what to explore:

### 1. Read the Documentation

- **[ARCHITECTURE.md](./ARCHITECTURE.md)** — Technical deep dive into graph layers, MCP servers, and data flow
- **[DEVELOPMENT.md](./DEVELOPMENT.md)** — Day-to-day development guide
- **[.claude/CLAUDE.md](../.claude/CLAUDE.md)** — Workspace architecture and agent roles

### 2. Explore the Agents

Each agent has a specific role:

| Agent | Role | Terminal Location |
|-------|------|-------------------|
| **Discuss** | Constitutional Guardian | `agents/discuss/` |
| **Plan** | Strategic Architect | `agents/plan/` |
| **Execute** | Operational Director | `agents/execute/` |
| **Mind** | Pattern Analyst | `agents/mind/` |
| **Health** | Health Intelligence | `agents/health/` |
| **Finance** | Financial Management | `agents/finance/` |

**Try running an agent:**

```bash
cd agents/discuss
claude .
```

This starts Claude Code in the Discuss agent workspace.

### 3. Make Your First Change

**Easy starter task:** Update the app name in the window title

1. Open `electron/main.js`
2. Find `mainWindow = new BrowserWindow({`
3. Change `title: 'ODEI'` to `title: 'ODEI - Your Name'`
4. Save and restart: `npm start`

### 4. Run Tests

```bash
# Unit tests
npm run test:unit

# E2E tests
npm run test:playwright

# All tests
npm run test:all
```

### 5. Explore the Graph

**Open Neo4j Browser** and try these queries:

```cypher
// See all node types
MATCH (n)
RETURN DISTINCT labels(n) as type, count(n) as count
ORDER BY count DESC

// View the constitutional foundation
MATCH (v:Value)-[:SUPPORTED_BY]->(p:Principle)
RETURN v.title, p.title

// Explore goal hierarchy
MATCH path = (v:Vision)<-[:SERVES]-(g:Goal)
RETURN path
LIMIT 25
```

---

## Development Workflow

Once you're set up, your typical workflow will be:

```bash
# 1. Start development mode
npm run dev

# 2. Make changes to code
# 3. Changes auto-reload (CSS) or restart Electron

# 4. Run tests
npm run test:unit

# 5. Commit changes
git add .
git commit -m "feat(ui): Add feature X"
```

**Pro tips:**

- Use **two terminals**: One for `npm run dev`, one for Git/tests
- Use **agent workspaces** for agent-specific work: `cd agents/discuss && claude .`
- Use **root workspace** for UI/infrastructure work: `cd /Users/ai/ODEI && claude .`

---

## Getting Help

### Documentation

- [ARCHITECTURE.md](./ARCHITECTURE.md) — System design
- [DEVELOPMENT.md](./DEVELOPMENT.md) — Dev workflow
- [TROUBLESHOOTING.md](./TROUBLESHOOTING.md) — Common issues
- [.claude/SCHEMA.md](../.claude/SCHEMA.md) — Graph schema reference

### Resources

- **Neo4j Cypher**: [neo4j.com/docs/cypher-manual](https://neo4j.com/docs/cypher-manual/current/)
- **Electron**: [electronjs.org/docs](https://www.electronjs.org/docs/latest/)
- **MCP Protocol**: [modelcontextprotocol.io](https://modelcontextprotocol.io)

### Support

- GitHub Issues: [Link to be added]
- Team Chat: [Link to be added]

---

## Summary

You've successfully:

- ✅ Installed all prerequisites
- ✅ Set up Neo4j database with Gen AI plugin
- ✅ Configured environment variables
- ✅ Built all MCP servers
- ✅ Bootstrapped foundation data
- ✅ Launched ODEI application
- ✅ Verified all components are working

**Welcome to the team. Let's build $500M together.**

---

_Last updated: 2025-12-25_
