# ODEI MCP Server Architecture

## Overview

ODEI uses the **Model Context Protocol (MCP)** to provide agents with specialized tools via external servers. This architecture separates concerns between different types of operations:

- **Canonical writes** → Single source of truth (odei-neo4j)
- **Agent-specific analytics** → Each agent has optimized read tools (discuss-neo4j, execute-neo4j, etc.)
- **Platform integrations** → Native system access (odei-apple for macOS Calendar)
- **Persistence** → Conversation history (odei-history)

```
┌─────────────────────────────────────────────────────────────┐
│                        ODEI Agents                          │
│  ┌─────────┐  ┌──────────┐  ┌─────────┐  ┌───────┐        │
│  │ Discuss │  │ Decisions│  │ Execute │  │ Mind  │        │
│  └────┬────┘  └─────┬────┘  └────┬────┘  └───┬───┘        │
└───────┼────────────┼────────────┼───────────┼─────────────┘
        │            │            │           │
        │            │            │           │
   ┌────▼────────────▼────────────▼───────────▼────┐
   │         MCP Server Layer (JSON-RPC)            │
   └────┬────────┬───────────┬──────────┬──────────┘
        │        │           │          │
┌───────▼──┐ ┌──▼────────┐ ┌▼────────┐ ┌▼──────────┐
│ odei-    │ │ agent-    │ │ odei-   │ │ odei-     │
│ neo4j    │ │ neo4j-*   │ │ apple   │ │ history   │
│          │ │           │ │         │ │           │
│ WRITES   │ │ ANALYTICS │ │ CALENDAR│ │ PERSIST   │
└──────────┘ └───────────┘ └─────────┘ └───────────┘
```

---

## MCP Server Types

### 1. Canonical Write Server: `odei-neo4j`

**Purpose:** Single source of truth for all Neo4j graph modifications.

**Location:** `/Users/ai/ODEI/servers/odei-neo4j`

**Technology:** TypeScript, compiled to `/dist/index.js`

**Environment:**

```json
{
  "NEO4J_URI": "bolt://127.0.0.1:7687",
  "NEO4J_USERNAME": "neo4j",
  "NEO4J_PASSWORD": "...",
  "NEO4J_DATABASE": "memory",
  "OPENAI_API_KEY": "...",
  "EMBEDDING_MODEL": "text-embedding-3-large"
}
```

**Key Characteristics:**

- ✅ Automatic embedding generation via Neo4j GenAI plugin
- ✅ CRUD + embedding enrichment
- ✅ Schema validation via Zod
- ✅ Provenance tracking
- ✅ Idempotency support
- ✅ Compatible with vector indexes / GenAI workflows

**Tools Provided:**

**Core CRUD Tools:**

```
odei.neo4j.<nodeType>.create.v1
odei.neo4j.<nodeType>.update.v1
odei.neo4j.node.update.v1
odei.neo4j.node.delete.v1
odei.neo4j.relationship.create.v1
odei.neo4j.<layer>.list.v1
odei.neo4j.graph.getAll.v1
odei.neo4j.schema.inventory.v1
odei.neo4j.context.add.v1
odei.neo4j.context.usage.v1
```

**Agent-Specific Tools:**

```
odei.neo4j.discuss.stats.v1
odei.neo4j.discuss.activeThreads.v1
odei.neo4j.decisions.backlog.v1
odei.neo4j.decisions.metrics.v1
odei.neo4j.execute.tasksToday.v1
odei.neo4j.execute.workSessions.v1
odei.neo4j.mind.recentInsights.v1
odei.neo4j.mind.patternStats.v1
```

**Node Types Supported:**

- **Foundation (Layer 1):** Value, Principle, Guardrail, Human, AI, Policy, Context
- **Vision (Layer 2):** Vision, Business, Goal, Season
- **Strategy (Layer 3):** Strategy, Objective, KeyResult, Initiative, Milestone, Risk
- **Tactics (Layer 4):** Project, Area, System, Process
- **Execution (Layer 5):** Decision, Task, TimeBlock, WorkSession, Action
- **Track (Layer 6):** Metric, Observation, Event, Signal
- **Mind (Layer 7):** Insight, Pattern, Evidence, Source, Note

**Build & Run:**

```bash
cd /Users/ai/ODEI/servers/odei-neo4j
npm run build
node dist/index.js
```

---

### 2. Agent-Specific Analytics Servers

Each agent has a specialized analytics server with AI-powered search and graph algorithms.

#### 2.1 `discuss-neo4j`

**Purpose:** Constitutional validation, goal alignment, workload assessment for Discuss agent.

**Location:** `/Users/ai/ODEI/agents/discuss/mcp-servers/neo4j`

**Environment:**

```json
{
  "NEO4J_URI": "bolt://127.0.0.1:7687",
  "NEO4J_USERNAME": "neo4j",
  "NEO4J_PASSWORD": "...",
  "NEO4J_DATABASE": "memory",
  "OPENAI_API_KEY": "sk-proj-...",
  "EMBEDDING_MODEL": "text-embedding-3-large",
  "EMBEDDING_DIMENSIONS": "3072"
}
```

**Tools Provided:**

```
discuss_semantic_search         — Vector similarity search (Values, Principles, Goals)
discuss_hybrid_search           — Combined text + vector search
discuss_goal_ladder             — Goal hierarchy traversal
discuss_workload_assess         — Capacity calculation from Tasks
discuss_graph_pagerank          — Influence analysis
discuss_graph_communities       — Cluster detection
discuss_graph_betweenness       — Bottleneck identification
discuss_graph_similar_nodes     — Find similar nodes by structure
discuss_graph_embeddings        — Generate graph-based embeddings
discuss_graph_shortest_path     — Path finding
discuss_expand_path             — Path expansion utilities
discuss_subgraph_nodes          — Subgraph extraction
discuss_concept_bridge          — Concept connection finder
discuss_aggregate_related       — Aggregate related node data
discuss_temporal_evolution      — Time-series analysis
discuss_pattern_detection       — Pattern recognition
discuss_cross_agent_query       — Inter-agent data queries
discuss_value_create            — Create Value nodes
discuss_principle_create        — Create Principle nodes
discuss_guardrail_create        — Create Guardrail nodes
discuss_vision_create           — Create Vision nodes
discuss_goal_create             — Create Goal nodes
discuss_memory_create           — Create Memory nodes
discuss_relationship_create     — Create relationships
discuss_graph_projection_create — Create graph projections
```

**Default Node Types for Search:**

- `Value`, `Principle`, `Guardrail`, `Goal`, `Memory`, `Vision`

**Run:**

```bash
cd /Users/ai/ODEI/agents/discuss/mcp-servers/neo4j
node index.js
```

#### 2.2 `execute-neo4j`

**Purpose:** Task execution tracking, progress monitoring, velocity analysis for Execute agent.

**Location:** `/Users/ai/ODEI/agents/execute/mcp-servers/neo4j`

**Environment:** Same as discuss-neo4j (requires OPENAI_API_KEY)

**Tools Provided:**

```
execute_semantic_search         — Vector search (Tasks, Projects, Systems)
execute_hybrid_search           — Combined search
execute_goal_ladder             — Goal → Project → Task chain
execute_workload_assess         — Capacity from pending Tasks
execute_graph_*                 — Graph algorithms (same as discuss)
execute_expand_path             — Path utilities
execute_subgraph_nodes          — Subgraph extraction
execute_concept_bridge          — Concept connection
execute_aggregate_related       — Aggregate related data
execute_temporal_evolution      — Time-series
execute_pattern_detection       — Pattern recognition
execute_cross_agent_query       — Inter-agent queries
```

**Default Node Types for Search:**

- `Task`, `Project`, `Area`, `System`, `Process`, `TimeBlock`, `WorkSession`

**Run:**

```bash
cd /Users/ai/ODEI/agents/execute/mcp-servers/neo4j
node index.js
```

#### 2.3 `decisions-neo4j`, `mind-neo4j`

Similar structure to above, each optimized for their respective agent's domain.

---

### 3. Platform Integration: `odei-apple`

**Purpose:** Apple Calendar integration via EventKit (macOS only).

**Location:** `/Users/ai/ODEI/servers/odei-apple`

**Technology:** Swift, compiled binary at `.build/release/odei-apple`

**Environment:** None (no credentials needed, uses macOS system authorization)

**Tools Provided:**

**Read-only (Discuss + Execute):**

```
odei.apple.calendar.window.v1   — Query events in time window
odei.apple.listCalendars.v1     — List available calendars
odei.apple.health.v1            — Check authorization status
odei.apple.context.add.v1       — Add context to calendar events
```

**Write (Execute only):**

```
odei.apple.createEvent.v1       — Create calendar event (supports dryRun)
odei.apple.updateEvent.v1       — Update event (supports dryRun)
odei.apple.deleteEvent.v1       — Delete event (supports dryRun)
```

**Authorization:**

- First run prompts: "Allow odei-apple to access Calendar?"
- User must approve in System Settings → Privacy & Security → Calendars
- Check status with `odei.apple.health.v1`

**Build & Run:**

```bash
cd /Users/ai/ODEI/servers/odei-apple
swift build -c release
.build/release/odei-apple
```

**Non-macOS Behavior:**

- All tools respond with `UNSUPPORTED_PLATFORM` error

---

### 4. Conversation Persistence: `odei-history`

**Purpose:** Store and retrieve agent conversation threads.

**Location:** `/Users/ai/ODEI/servers/odei-history`

**Technology:** TypeScript, compiled to `/dist/index.js`

**Environment:**

```json
{
  "STORAGE_PATH": "/Users/ai/ODEI/data/history"
}
```

**Tools Provided:**

```
odei.history.threads.list.v1           — List conversation threads
odei.history.threads.get.v1            — Get specific thread
odei.history.threads.create.v1         — Create new thread
odei.history.threads.update.v1         — Update thread
odei.history.threads.appendMessage.v1  — Add message to thread
odei.history.conversations.create.v1   — Create conversation
odei.history.conversations.query.v1    — Query conversations
odei.history.threads.search.v1         — Search threads
odei.history.threads.delete.v1         — Delete thread
```

**Used by:**

- Electron app for UI conversation loading
- Agents for context persistence across sessions

**Run:**

```bash
cd /Users/ai/ODEI/servers/odei-history
npm run build
node dist/index.js
```

---

## Agent MCP Configuration

Each agent has its own MCP configuration file at the agent root:

```
/Users/ai/ODEI/agents/{agent}/.mcp.json
```

**Example: Discuss Agent Configuration**

```json
{
  "mcpServers": {
    "odei-apple": {
      "command": "/Users/ai/ODEI/servers/odei-apple/.build/release/odei-apple",
      "args": [],
      "toolAllowList": ["odei.apple.calendar.window.v1", "odei.apple.listCalendars.v1", "odei.apple.health.v1"]
    },
    "discuss-neo4j": {
      "command": "node",
      "args": ["/Users/ai/ODEI/agents/discuss/mcp-servers/neo4j/index.js"],
      "env": {
        "NEO4J_URI": "bolt://127.0.0.1:7687",
        "NEO4J_USERNAME": "neo4j",
        "NEO4J_PASSWORD": "...",
        "NEO4J_DATABASE": "memory",
        "OPENAI_API_KEY": "sk-proj-...",
        "EMBEDDING_MODEL": "text-embedding-3-large",
        "EMBEDDING_DIMENSIONS": "3072"
      },
      "toolAllowList": [
        "discuss_semantic_search",
        "discuss_hybrid_search",
        "discuss_goal_ladder",
        "discuss_workload_assess",
        "discuss_graph_pagerank",
        "discuss_graph_communities",
        "discuss_graph_betweenness",
        "discuss_graph_similar_nodes",
        "discuss_graph_embeddings",
        "discuss_graph_shortest_path",
        "discuss_expand_path",
        "discuss_subgraph_nodes",
        "discuss_concept_bridge",
        "discuss_aggregate_related",
        "discuss_temporal_evolution",
        "discuss_pattern_detection",
        "discuss_cross_agent_query"
      ]
    },
    "odei-neo4j": {
      "command": "node",
      "args": ["/Users/ai/ODEI/servers/odei-neo4j/dist/index.js"],
      "env": {
        "NEO4J_URI": "bolt://127.0.0.1:7687",
        "NEO4J_USERNAME": "neo4j",
        "NEO4J_PASSWORD": "...",
        "NEO4J_DATABASE": "memory"
      },
      "toolAllowList": [
        "odei.neo4j.value.create.v1",
        "odei.neo4j.value.update.v1",
        "odei.neo4j.principle.create.v1",
        "odei.neo4j.principle.update.v1",
        "odei.neo4j.guardrail.create.v1",
        "odei.neo4j.guardrail.update.v1",
        "odei.neo4j.vision.create.v1",
        "odei.neo4j.vision.update.v1",
        "odei.neo4j.goal.create.v1",
        "odei.neo4j.goal.update.v1",
        "odei.neo4j.relationship.create.v1",
        "odei.neo4j.foundation.list.v1",
        "odei.neo4j.vision.list.v1",
        "odei.neo4j.graph.getAll.v1",
        "odei.neo4j.schema.inventory.v1"
      ]
    }
  },
  "comment": "Discuss agent: read/search via discuss-neo4j, canonical writes via odei-neo4j."
}
```

**Security Note:** Real deployments should use environment variables or secrets management - hardcoded keys shown here for development convenience only.

---

## Agent Access Matrix

| Agent            | odei-apple     | agent-neo4j                | odei-neo4j                  | odei-history |
| ---------------- | -------------- | -------------------------- | --------------------------- | ------------ |
| **Discuss**      | READ (3 tools) | discuss-neo4j (17 tools)   | Foundation+Vision writes    | ❌           |
| **Decisions**    | ❌             | decisions-neo4j (17 tools) | Strategy+Execution writes   | ❌           |
| **Execute**      | FULL (6 tools) | execute-neo4j (17 tools)   | Tactics+Execution writes    | ❌           |
| **Mind**         | ❌             | mind-neo4j (17 tools)      | Track+Mind writes           | ❌           |
| **Electron App** | ❌             | ❌                         | READ (list, getAll, schema) | FULL         |

---

## How to Add an MCP Server to an Agent

### Step 1: Identify the Server

Determine which MCP server you want to add:

- **odei-neo4j** — For canonical graph writes
- **odei-apple** — For calendar access
- **odei-history** — For conversation persistence
- **agent-neo4j** — Already included per agent
- **Custom server** — Build your own (see below)

### Step 2: Locate Agent Configuration

Navigate to the agent's MCP configuration file:

```bash
cd /Users/ai/ODEI/agents/{agent_name}
# File: .mcp.json (NOT .claude/mcp.json!)
```

**Important:** Claude Code reads `.mcp.json` at the agent root, NOT `.claude/mcp.json`.

### Step 3: Add Server Entry

Edit `/Users/ai/ODEI/agents/{agent_name}/.mcp.json`:

```json
{
  "mcpServers": {
    "server-name": {
      "command": "/path/to/server/binary-or-node",
      "args": ["optional", "args"],
      "env": {
        "ENV_VAR": "value"
      },
      "toolAllowList": ["tool.name.v1", "another.tool.v1"]
    }
  }
}
```

**Example: Add odei-apple to Decisions agent**

```json
{
  "mcpServers": {
    "odei-apple": {
      "command": "/Users/ai/ODEI/servers/odei-apple/.build/release/odei-apple",
      "args": [],
      "toolAllowList": ["odei.apple.calendar.window.v1", "odei.apple.listCalendars.v1", "odei.apple.health.v1"]
    },
    "decisions-neo4j": {
      // ... existing config
    },
    "odei-neo4j": {
      // ... existing config
    }
  }
}
```

### Step 4: Specify Tool Allow List

The `toolAllowList` restricts which tools the agent can access. This is critical for:

- **Security:** Prevent agents from calling unintended tools
- **Minimalist access:** Only grant tools the agent actually needs
- **Performance:** Reduces tool discovery overhead

**To find available tools:**

```bash
# For TypeScript servers
cat /Users/ai/ODEI/servers/odei-neo4j/src/tools/index.ts

# For Node.js servers
grep "case '" /Users/ai/ODEI/agents/discuss/mcp-servers/neo4j/index.js

# For Swift servers
grep "@Tool" /Users/ai/ODEI/servers/odei-apple/Sources/*.swift
```

### Step 5: Configure Environment Variables

Different servers require different environment variables:

**odei-neo4j:**

```json
"env": {
  "NEO4J_URI": "bolt://127.0.0.1:7687",
  "NEO4J_USERNAME": "neo4j",
  "NEO4J_PASSWORD": "your-password",
  "NEO4J_DATABASE": "memory"
}
```

**agent-neo4j (with embeddings):**

```json
"env": {
  "NEO4J_URI": "bolt://127.0.0.1:7687",
  "NEO4J_USERNAME": "neo4j",
  "NEO4J_PASSWORD": "your-password",
  "NEO4J_DATABASE": "memory",
  "OPENAI_API_KEY": "sk-proj-...",
  "EMBEDDING_MODEL": "text-embedding-3-large",
  "EMBEDDING_DIMENSIONS": "3072"
}
```

**odei-apple:**

```json
// No env vars needed (uses macOS system authorization)
```

**odei-history:**

```json
"env": {
  "STORAGE_PATH": "/Users/ai/ODEI/data/history"
}
```

### Step 6: Restart Agent

After modifying `.mcp.json`, restart the agent:

**From Electron app:**

- Close agent window
- Reopen agent (File → Open Agent → {agent_name})

**From CLI:**

```bash
cd /Users/ai/ODEI/agents/{agent_name}
claude .
```

**From parent Electron process:**

- Restart the entire application to reload all agents

### Step 7: Verify Server Connection

Open the agent's MCP dialog to verify the server is connected:

1. **In agent chat**, type `/mcp` or use menu: Tools → MCP Servers
2. You should see:
   ```
   ❯ 1. server-name            ✔ connected
   Project config: /Users/ai/ODEI/agents/{agent}/.mcp.json
   ```
3. Click "Show Tools" to verify tool list matches your `toolAllowList`

### Step 8: Test Tool Usage

Try calling a tool from the agent:

```
User: Call odei.apple.listCalendars.v1
Agent: [Calls MCP tool and returns calendar list]
```

Or let the agent use it naturally:

```
User: What's on my calendar tomorrow?
Agent: [Automatically calls odei.apple.calendar.window.v1]
```

---

## Common Configuration Patterns

### Pattern 1: Read-Only Calendar Access (Discuss)

```json
{
  "mcpServers": {
    "odei-apple": {
      "command": "/Users/ai/ODEI/servers/odei-apple/.build/release/odei-apple",
      "args": [],
      "toolAllowList": ["odei.apple.calendar.window.v1", "odei.apple.listCalendars.v1", "odei.apple.health.v1"]
    }
  }
}
```

**Use case:** Workload assessment, capacity calculation

### Pattern 2: Full Calendar Access (Execute)

```json
{
  "mcpServers": {
    "odei-apple": {
      "command": "/Users/ai/ODEI/servers/odei-apple/.build/release/odei-apple",
      "args": [],
      "toolAllowList": [
        "odei.apple.calendar.window.v1",
        "odei.apple.listCalendars.v1",
        "odei.apple.health.v1",
        "odei.apple.createEvent.v1",
        "odei.apple.updateEvent.v1",
        "odei.apple.deleteEvent.v1"
      ]
    }
  }
}
```

**Use case:** Schedule optimization, TimeBlock management

### Pattern 3: Layer-Specific Graph Access (Discuss)

```json
{
  "mcpServers": {
    "odei-neo4j": {
      "command": "node",
      "args": ["/Users/ai/ODEI/servers/odei-neo4j/dist/index.js"],
      "env": { "NEO4J_URI": "...", "NEO4J_USERNAME": "...", "NEO4J_PASSWORD": "...", "NEO4J_DATABASE": "memory" },
      "toolAllowList": [
        "odei.neo4j.value.create.v1",
        "odei.neo4j.value.update.v1",
        "odei.neo4j.principle.create.v1",
        "odei.neo4j.principle.update.v1",
        "odei.neo4j.guardrail.create.v1",
        "odei.neo4j.guardrail.update.v1",
        "odei.neo4j.vision.create.v1",
        "odei.neo4j.vision.update.v1",
        "odei.neo4j.goal.create.v1",
        "odei.neo4j.goal.update.v1",
        "odei.neo4j.relationship.create.v1",
        "odei.neo4j.foundation.list.v1",
        "odei.neo4j.vision.list.v1",
        "odei.neo4j.graph.getAll.v1",
        "odei.neo4j.schema.inventory.v1"
      ]
    }
  }
}
```

**Use case:** Discuss only writes to foundation (1) and vision (2) layers

### Pattern 4: Analytics + Writes (Standard Agent Pattern)

```json
{
  "mcpServers": {
    "agent-neo4j": {
      "command": "node",
      "args": ["/Users/ai/ODEI/agents/{agent}/mcp-servers/neo4j/index.js"],
      "env": {
        "NEO4J_URI": "...",
        "NEO4J_USERNAME": "...",
        "NEO4J_PASSWORD": "...",
        "NEO4J_DATABASE": "memory",
        "OPENAI_API_KEY": "sk-proj-...",
        "EMBEDDING_MODEL": "text-embedding-3-large",
        "EMBEDDING_DIMENSIONS": "3072"
      },
      "toolAllowList": [
        "{agent}_semantic_search",
        "{agent}_hybrid_search",
        "{agent}_goal_ladder",
        "{agent}_workload_assess",
        "{agent}_graph_pagerank",
        "{agent}_graph_communities",
        "{agent}_graph_betweenness",
        "{agent}_graph_similar_nodes",
        "{agent}_graph_embeddings",
        "{agent}_graph_shortest_path",
        "{agent}_expand_path",
        "{agent}_subgraph_nodes",
        "{agent}_concept_bridge",
        "{agent}_aggregate_related",
        "{agent}_temporal_evolution",
        "{agent}_pattern_detection",
        "{agent}_cross_agent_query"
      ]
    },
    "odei-neo4j": {
      "command": "node",
      "args": ["/Users/ai/ODEI/servers/odei-neo4j/dist/index.js"],
      "env": { "NEO4J_URI": "...", "NEO4J_USERNAME": "...", "NEO4J_PASSWORD": "...", "NEO4J_DATABASE": "memory" },
      "toolAllowList": [
        "odei.neo4j.{nodeType}.create.v1",
        "odei.neo4j.{nodeType}.update.v1",
        "odei.neo4j.relationship.create.v1",
        "odei.neo4j.{layer}.list.v1",
        "odei.neo4j.graph.getAll.v1",
        "odei.neo4j.schema.inventory.v1"
      ]
    }
  }
}
```

**Use case:** Most agents (Discuss, Decisions, Execute, Mind)

---

## Lazy MCP Loading

MCP servers are **NOT started** when the agent launches. They are started **on-demand** when:

1. **Agent calls a tool** from that server for the first time
2. **User requests MCP server list** (via `/mcp` command)

**Benefits:**

- Faster agent startup (no waiting for all MCP servers)
- Lower resource usage (only active servers consume memory)
- Cleaner logs (only relevant server output)

**Monitoring:**

- Electron app logs show: `[AgentManager] Starting MCP server: discuss-neo4j`
- Check connection status via agent MCP dialog: `✔ connected` or `✘ error`

---

## Known Limitations

### 1. Cross-Server Dependencies

**Problem:**

- Agent-neo4j servers cannot call odei-neo4j tools (separate processes)
- No inter-MCP-server communication protocol
- Agents must orchestrate multi-server workflows manually

**Example:**

```
Agent:
1. Create Value via odei-neo4j
2. Wait for response (node ID)
3. Generate embedding via discuss-neo4j
4. Update node with embedding via odei-neo4j
```

**Future Solution:**

- MCP server-to-server communication
- Or: Unified server eliminates need

### 2. Platform-Specific Tools (odei-apple)

**Problem:**

- `odei-apple` only works on macOS
- Non-macOS environments get `UNSUPPORTED_PLATFORM` errors
- Agents must handle platform detection

**Workaround:**

```json
// In non-macOS environments, omit odei-apple from .mcp.json
{
  "mcpServers": {
    // "odei-apple": { ... }  ← Comment out or remove
    "discuss-neo4j": { ... },
    "odei-neo4j": { ... }
  }
}
```

### 3. Configuration File Location Confusion

**Problem:**

- `.claude/mcp.json` exists but is NOT read by Claude Code
- Actual config is `.mcp.json` at agent root
- Easy to edit wrong file

**Solution:**

- ALWAYS edit `/Users/ai/ODEI/agents/{agent}/.mcp.json`
- Ignore `/Users/ai/ODEI/agents/{agent}/.claude/mcp.json`

---

## Troubleshooting

### Server Not Connecting

**Symptoms:**

- Agent MCP dialog shows `✘ error` or `⏳ loading` forever
- Tools not available in agent

**Diagnosis:**

```bash
# Check if server binary exists
ls -la /Users/ai/ODEI/servers/odei-apple/.build/release/odei-apple

# For Node servers, check if built
ls -la /Users/ai/ODEI/servers/odei-neo4j/dist/index.js

# Check Electron app logs
tail -f /Users/ai/ODEI/electron/logs/main.log
```

**Fix:**

1. Build the server:
   ```bash
   cd /Users/ai/ODEI/servers/odei-neo4j
   npm run build
   ```
2. Check `.mcp.json` command path is correct
3. Restart agent

### Tools Not Appearing

**Symptoms:**

- Server shows `✔ connected` but tools missing

**Diagnosis:**

- Check `toolAllowList` in `.mcp.json`
- Verify tool names match server's exported tools

**Fix:**

```bash
# List available tools
grep "case '" /Users/ai/ODEI/agents/discuss/mcp-servers/neo4j/index.js
```

Add missing tools to `toolAllowList` and restart.

### Environment Variables Not Working

**Symptoms:**

- Server crashes with "Missing env var" error
- Embeddings fail with "OPENAI_API_KEY not set"

**Diagnosis:**

- Check `env` section in `.mcp.json`
- Verify values are strings (not null/undefined)

**Fix:**

```json
{
  "mcpServers": {
    "discuss-neo4j": {
      "env": {
        "OPENAI_API_KEY": "sk-proj-ACTUAL-KEY-HERE" // NOT null or ""
      }
    }
  }
}
```

### Semantic Search Returns Empty

**Symptoms:**

- `discuss_semantic_search` returns `[]` for known nodes

**Diagnosis:**

- Check if embeddings exist:
  ```cypher
  MATCH (n:Value)
  RETURN n.title, n.embedding IS NOT NULL AS hasEmbedding
  ```

**Fix:**

```bash
# Generate embeddings manually
User: Run batchUpdateEmbeddings for all Value nodes
Agent: [Calls tool to generate embeddings]
```

---

## Creating a New MCP Server

### Step 1: Choose Technology

- **TypeScript** (for complex logic, schema validation)
- **JavaScript** (for simpler servers)
- **Swift** (for macOS-specific features)
- **Python** (for ML/AI tools)

### Step 2: Scaffold Server

**TypeScript Example:**

```bash
mkdir -p /Users/ai/ODEI/servers/my-server
cd /Users/ai/ODEI/servers/my-server
npm init -y
npm install @modelcontextprotocol/sdk zod neo4j-driver
```

**Create `src/index.ts`:**

```typescript
import { Server } from '@modelcontextprotocol/sdk/server/index.js';
import { StdioServerTransport } from '@modelcontextprotocol/sdk/server/stdio.js';
import { CallToolRequestSchema, ListToolsRequestSchema } from '@modelcontextprotocol/sdk/types.js';

const server = new Server({ name: 'my-server', version: '1.0.0' }, { capabilities: { tools: {} } });

server.setRequestHandler(ListToolsRequestSchema, async () => ({
  tools: [
    {
      name: 'my.tool.v1',
      description: 'My custom tool',
      inputSchema: {
        type: 'object',
        properties: {
          input: { type: 'string' },
        },
        required: ['input'],
      },
    },
  ],
}));

server.setRequestHandler(CallToolRequestSchema, async (request) => {
  const { name, arguments: args } = request.params;

  if (name === 'my.tool.v1') {
    return {
      content: [
        {
          type: 'text',
          text: `You said: ${args.input}`,
        },
      ],
    };
  }

  throw new Error(`Unknown tool: ${name}`);
});

async function main() {
  const transport = new StdioServerTransport();
  await server.connect(transport);
  console.error('My Server running on stdio');
}

main().catch(console.error);
```

### Step 3: Build

```bash
npm run build
# Output: dist/index.js
```

### Step 4: Test Locally

```bash
node dist/index.js
# Server should print: "My Server running on stdio"
# Press Ctrl+C to stop
```

### Step 5: Add to Agent

Edit `/Users/ai/ODEI/agents/discuss/.mcp.json`:

```json
{
  "mcpServers": {
    "my-server": {
      "command": "node",
      "args": ["/Users/ai/ODEI/servers/my-server/dist/index.js"],
      "env": {},
      "toolAllowList": ["my.tool.v1"]
    }
  }
}
```

### Step 6: Test from Agent

Restart agent, then:

```
User: Call my.tool.v1 with input "hello"
Agent: [Returns "You said: hello"]
```

---

## Best Practices

### 1. Minimalist Tool Access

Only grant tools an agent actually needs:

❌ **Bad:**

```json
"toolAllowList": ["*"]  // Gives access to ALL tools
```

✅ **Good:**

```json
"toolAllowList": [
  "odei.apple.calendar.window.v1",
  "odei.apple.listCalendars.v1"
]
```

### 2. Layer-Appropriate Writes

Respect agent boundaries:

- **Discuss** → foundation (1) + vision (2) only
- **Decisions** → strategy (3) + execution (5, Decision nodes) only
- **Execute** → tactics (4) + execution (5, non-Decision) only
- **Mind** → track (6) + mind (7) only

### 3. Environment Variable Security

Never commit secrets to Git:

❌ **Bad:**

```json
{
  "env": {
    "OPENAI_API_KEY": "sk-proj-REAL-KEY-HERE" // ← Committed to repo!
  }
}
```

✅ **Good:**

```bash
# Use environment variables or .env files (gitignored)
export OPENAI_API_KEY="sk-proj-..."
```

Then in `.mcp.json`:

```json
{
  "env": {
    "OPENAI_API_KEY": "${OPENAI_API_KEY}" // ← Reads from parent env
  }
}
```

### 4. Lazy Loading Optimization

Design servers for fast startup:

- Defer expensive initialization (DB connections, model loading)
- Only connect when first tool called
- Reuse connections across tool calls

### 5. Error Handling

Always provide clear error messages:

```typescript
if (!args.nodeId) {
  throw new Error('nodeId is required for odei.neo4j.value.update.v1');
}
```

Agents can't debug cryptic errors!

---

## Summary

ODEI's MCP architecture provides:

- ✅ **Separation of concerns:** Read vs Write, Analytics vs CRUD
- ✅ **Agent-specific optimization:** Each agent sees only relevant tools
- ✅ **Lazy loading:** Servers start on-demand, not at launch
- ✅ **Minimalist access:** Tools restricted via allowList
- ✅ **Platform integration:** Native system access (Calendar, etc.)

**Key Files:**

- `/Users/ai/ODEI/agents/{agent}/.mcp.json` — Agent config
- `/Users/ai/ODEI/servers/odei-neo4j/` — Canonical writes
- `/Users/ai/ODEI/servers/odei-apple/` — Calendar integration
- `/Users/ai/ODEI/agents/{agent}/mcp-servers/neo4j/` — Analytics

**Next Steps:**

- Add calendar to agents: See CALENDAR_INTEGRATION.md
- Build custom MCP server: Follow "Creating a New MCP Server" above
- Fix embeddings issue: Add OPENAI_API_KEY to odei-neo4j (future)
