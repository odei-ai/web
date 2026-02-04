# ODEI MCP Server Ecosystem Complete Inventory

## Summary

This directory contains a comprehensive analysis of the ODEI MCP server ecosystem. There are **5 MCP servers** providing **100+ tools** across **2 workspace levels** (root + 4 agents).

## Quick Start

**New to ODEI MCP?** Start here:

1. Read: **MCP_QUICK_REFERENCE.md** (5 min overview)
2. Visualize: **MCP_ARCHITECTURE_DIAGRAM.txt** (understand relationships)
3. Deep dive: **MCP_SERVER_INVENTORY.md** (complete reference)

## Documentation Files

### Primary References (Latest & Most Useful)

**MCP_SERVER_INVENTORY.md** (974 lines, comprehensive)

- Complete specifications for all 5 servers
- All 100+ tools documented with actual functionality
- Technology stacks, authentication, performance metrics
- Codebase structure and file locations
- Design principles and debugging guides
- **Status:** Complete, up-to-date as of 2025-11-05

**MCP_QUICK_REFERENCE.md** (practical guide)

- Top 10 most important tools with examples
- Common operations (search, create, update, check capacity)
- Workspace architecture and AllowList patterns
- Performance notes and credential management
- **Best for:** Practical usage, quick lookup

**MCP_ARCHITECTURE_DIAGRAM.txt** (visual)

- ASCII diagrams showing all server relationships
- Workspace access matrix
- Data flow diagrams
- 7-layer ODEI model structure
- Deployment architecture
- **Best for:** Understanding system design

### Supporting References (Previous Analyses)

**MCP_CONSOLIDATION_EXECUTIVE_SUMMARY.md**

- High-level overview of MCP consolidation work
- Historical context on server unification

**MCP_ARCHITECTURE_ANALYSIS_REPORT.md**

- Detailed architectural analysis
- Cross-server dependencies and integration patterns

**MCP_CONSOLIDATION_ANALYSIS.md**

- Previous consolidation efforts and their impact
- Historical technical decisions

**MCP_FIX_CHECKLIST.md**

- Known issues and fixes applied to servers
- Implementation status tracking

**MCP_INTEGRATION_TEST_INDEX.md**

- Test plans for MCP functionality
- Integration test scenarios

## The 5 MCP Servers at a Glance

### 1. odei-neo4j (Node.js) - The Brain

**Purpose:** Graph database for ODEI's 7-layer knowledge model

**Key Tools (62+):**

- Layer CRUD (Foundation → Mind)
- Node type operations (Value, Goal, Task, Insight, etc.)
- Semantic search (14ms latency)
- Graph algorithms (PageRank, communities, shortest path)
- Agent dashboards
- Infrastructure (backup, cache, context)

**Used by:** All agents + root workspace

### 2. odei-history (Node.js) - The Archive

**Purpose:** Conversation history persistence

**Key Tools (9):**

- Conversation CRUD
- Thread management
- Content search
- Message appending

**Used by:** Root + Mind agent

### 3. odei-miro (Node.js) - The Whiteboard

**Purpose:** Visual collaboration via mind maps

**Key Tools (3):**

- Create mind maps
- Add nodes
- List boards

**Used by:** All agents

### 4. odei-apple (Swift) - The Calendar

**Purpose:** macOS calendar integration

**Key Tools (3+):**

- Query calendar events
- List calendars
- Health data (stub)

**Used by:** Discuss agent
**Platform:** macOS only

### 5. excalidraw-mcp (Node.js) - The Canvas

**Purpose:** Diagram creation and manipulation

**Key Tools (12):**

- Element management
- Grouping and alignment
- Batch operations
- Locking elements

**Used by:** Root workspace

## Workspace Architecture

### Root Workspace

Location: `/Users/ai/ODEI/.claude/mcp.json`

- Full access to all 5 servers
- Used for: Electron app, system-wide operations

### Agent Workspaces

- **Discuss** (`agents/discuss/.claude/mcp.json`): 59 Neo4j tools + calendar + Miro
- **Decisions** (`agents/decisions/.claude/mcp.json`): Restricted Neo4j + Miro
- **Execute** (`agents/execute/.claude/mcp.json`): Restricted Neo4j + Miro
- **Mind** (`agents/mind/.claude/mcp.json`): Restricted Neo4j + History + Miro

Each agent uses `toolAllowList` to restrict access to relevant capabilities.

## 7-Layer ODEI Model (in Neo4j)

1. **Foundation** - Values, Principles, Guardrails (what we believe)
2. **Vision** - Visions, Goals (where we're going)
3. **Strategy** - Objectives, KeyResults, Initiatives (how to get there)
4. **Tactics** - Projects, Areas, Systems, Processes (what we're building)
5. **Execution** - Tasks, Decisions, WorkSessions (what we're doing)
6. **Track** - Metrics, Observations, Events (what's happening)
7. **Mind** - Insights, Patterns, Evidence (what we learned)

## Key Capabilities

1. **Semantic Search** (14ms latency)
   - Vector embeddings + keyword matching + graph expansion
   - Personalized ranking based on user patterns
   - 700x faster than ChatGPT memory

2. **Graph Algorithms**
   - PageRank for influence analysis
   - Community detection for clustering
   - Shortest path for dependencies
   - Similarity for pattern replication

3. **Calendar Integration**
   - Query events for capacity assessment
   - Support for scheduling and time management

4. **Visual Planning**
   - Miro mind maps for goal/strategy visualization
   - Excalidraw diagrams for system design

5. **Conversation Archival**
   - Full history persistence
   - Content search and retrieval
   - Pattern analysis support

## Performance Metrics

| Server         | Latency                     | Throughput   | Memory |
| -------------- | --------------------------- | ------------ | ------ |
| odei-neo4j     | 14ms (search), 50ms (write) | 100 ops/sec  | 400MB  |
| odei-history   | <5ms                        | 1000 ops/sec | 50MB   |
| odei-miro      | 200-500ms (API)             | 10 ops/sec   | 30MB   |
| odei-apple     | <5ms                        | 100 ops/sec  | 20MB   |
| excalidraw-mcp | 50-200ms                    | 50 ops/sec   | 80MB   |

## Most Important Tools (Top 10)

1. `odei.neo4j.hybrid.search.v1` - Universal search
2. `odei.neo4j.workload.assess.v1` - Capacity checking
3. `odei.neo4j.{type}.create.v1` - Create nodes
4. `odei.neo4j.{type}.update.v1` - Edit nodes
5. `odei.neo4j.foundation.list.v2` - Session context
6. `odei.neo4j.vision.list.v2` - Goal hierarchy
7. `odei.neo4j.relationship.create.v1` - Link nodes
8. `odei.apple.calendar.window.v1` - Calendar queries
9. `odei.miro.mindmap.create.v1` - Visual planning
10. `odei.history.threads.search.v1` - History search

## Common Operations

### Find Something

```json
odei.neo4j.hybrid.search.v1({
  "query": "what you're looking for",
  "nodeTypes": ["Goal", "Value"],
  "topK": 10,
  "graphDepth": 3
})
```

### Create a Goal

```json
odei.neo4j.goal.create.v1({
  "data": {
    "title": "Ship MVP",
    "description": "...",
    "status": "active"
  },
  "provenance": {"actor": "human", "module": "discuss"}
})
```

### Check Capacity

```json
odei.neo4j.workload.assess.v1({
  "days": 7
})
```

## File Organization

```
/Users/ai/ODEI/
├── servers/
│   ├── odei-neo4j/          (Graph database)
│   ├── odei-history/        (Conversation storage)
│   ├── odei-miro/           (Mind maps)
│   ├── odei-apple/          (Calendar)
│   └── excalidraw-mcp/      (Diagrams)
│
├── agents/
│   ├── discuss/.claude/mcp.json
│   ├── decisions/.claude/mcp.json
│   ├── execute/.claude/mcp.json
│   └── mind/.claude/mcp.json
│
├── .claude/mcp.json         (Root workspace)
└── data/history.sqlite      (Conversation database)
```

## Credentials & Secrets

| Server         | Credentials                                   |
| -------------- | --------------------------------------------- |
| odei-neo4j     | NEO4J_URI, USERNAME, PASSWORD, OPENAI_API_KEY |
| odei-history   | ODEI_HISTORY_DB_PATH                          |
| odei-miro      | MIRO_ACCESS_TOKEN, MIRO_DEFAULT_BOARD_ID      |
| odei-apple     | macOS EventKit (system auth)                  |
| excalidraw-mcp | EXPRESS_SERVER_URL (optional)                 |

All stored in `.claude/mcp.json` environment variables.

## Debugging & Support

**Health Check:**

```bash
# Test odei-neo4j
echo '{"jsonrpc":"2.0","id":1,"method":"health.ping"}' | \
  node /Users/ai/ODEI/servers/odei-neo4j/dist/index.js
```

**Common Issues:**

1. Neo4j connection fails → Check NEO4J_URI, USERNAME, PASSWORD
2. Miro token invalid → Run OAuth token refresh
3. Apple returns UNSUPPORTED_PLATFORM → Not running on macOS
4. Embeddings API errors → Check OPENAI_API_KEY quota

## Documentation Index

For specific topics, see:

- **Complete Inventory:** MCP_SERVER_INVENTORY.md
- **Quick Reference:** MCP_QUICK_REFERENCE.md
- **Architecture Diagrams:** MCP_ARCHITECTURE_DIAGRAM.txt
- **Consolidation History:** MCP_CONSOLIDATION_EXECUTIVE_SUMMARY.md
- **Implementation Status:** MCP_FIX_CHECKLIST.md
- **Integration Tests:** MCP_INTEGRATION_TEST_INDEX.md

## Key Design Principles

1. **Centralized Graph** - Neo4j is source of truth for all structural data
2. **Service Separation** - Each server has single responsibility
3. **Agent-Specific Access** - AllowList controls what each agent can access
4. **Semantic Foundation** - All search goes through Neo4j hybrid search
5. **Environment Isolation** - Credentials via ${VAR} references, never hardcoded

## Next Steps

- **New User:** Read MCP_QUICK_REFERENCE.md
- **System Admin:** Review MCP_SERVER_INVENTORY.md
- **Developer:** Check server implementations in servers/ directory
- **Troubleshooting:** See MCP_FIX_CHECKLIST.md

---

**Document Version:** 1.0  
**Last Updated:** 2025-11-05  
**Total Servers:** 5  
**Total Tools:** 100+  
**Workspace Configurations:** 5 (1 root + 4 agents)
