# ODEI Terminology Glossary

## Core Concepts

### Architecture Terminology

**3-Tier Agent System** (Execution Architecture)
The multi-layered execution model for ODEI agents:

- **Tier 1: Multi-Agent** — 4 autonomous Claude Code agents (discuss, plan, execute, mind) with dedicated workspaces + MCP configs.
- **Tier 2: Subagents** — Task-based behaviors defined in `.claude/agents/{name}.md`, executed through Claude Code’s Task tool in isolated mini-contexts.
- **Tier 3: Agent Prompts** — Each workspace’s `CLAUDE.md`, which encodes identity, guardrails, session-start protocol, and subagent triggers.

⚠️ Use "tier" not "layer" to distinguish from data model layers.

**7-Layer Data Model** (Neo4j Graph Structure)
The hierarchical graph database schema representing ODEI's constitutional memory:

1. **Foundation Layer** - Core values, principles, constitutional boundaries
2. **Vision Layer** - Long-term goals, aspirations, life direction
3. **Strategy Layer** - Multi-month plans, strategic initiatives
4. **Tactics Layer** - Week-to-month level execution plans
5. **Execution Layer** - Daily tasks, immediate actions
6. **Track Layer** - Progress metrics, performance indicators
7. **Mind Layer** - Insights, patterns, retrospective analysis

Each layer contains specific node types (e.g., Principle, Goal, Initiative, Task) and relationship types that connect nodes within and across layers.

**8-Stage Memory API** (Retrieval Stages)
The ordered stages used by `memory.retrieve.v1` tool for contextual memory retrieval:

1. **identity** - User profile, preferences, personal context
2. **temporal** - Time-based context (current time, date, schedule)
3. **strategic** - Active goals, initiatives, long-term plans
4. **operational** - Current tasks, immediate work context
5. **analytics** - Patterns, trends, performance metrics
6. **episodic** - Past conversations, historical events
7. **conversational** - Recent dialogue context
8. **procedural** - How-to knowledge, methodology references

Stages can be selectively enabled/disabled based on retrieval needs.

---

## Agent Names and Roles

### The Four Autonomous Agents

**discuss** - Constitutional Guardian & Life Partner

- **Role:** Validates alignment with constitutional principles, engages in philosophical dialogue
- **Workspace:** `agents/discuss/`
- **MCP Servers:** `odei-neo4j` (semantic search, constitutional validation)
- **Key Responsibilities:** Ethical oversight, value alignment, conversational support

**plan** - Strategic Visualization & Decision Architecture

- **Role:** Strategic planning, initiative decomposition, decision visualization
- **Workspace:** `agents/plan/`
- **MCP Servers:** `odei-neo4j` (strategic queries), `odei-miro` (mind map visualization)
- **Key Responsibilities:** Goal setting, strategic planning, Miro board management
- **Historical Note:** Previously called "decisions" in older documentation

**execute** - Tactical Lead & Workload Manager

- **Role:** Task execution, workload management, operational coordination
- **Workspace:** `agents/execute/`
- **MCP Servers:** `odei-neo4j` (task queries, execution tracking)
- **Key Responsibilities:** Daily task management, execution tracking, resource allocation

**mind** - Pattern Intelligence & Analytics

- **Role:** Pattern detection, retrospective analysis, system optimization
- **Workspace:** `agents/mind/`
- **MCP Servers:** `odei-neo4j` (analytics queries), `odei-history` (conversation analysis)
- **Key Responsibilities:** Insight extraction, performance analysis, continuous improvement

---

## Memory Concepts

### Memory Architecture Overview

ODEI's memory system has three distinct aspects that are often confused:

**10 Memory Types** (Design Concept)
The original conceptual framework for memory categorization:

- Episodic, Semantic, Procedural, Working, Prospective, Constitutional, Conversational, Contextual, Relational, Metacognitive

This is the theoretical model that informed the implementation.

**8 Retrieval Stages** (Implementation)
The practical stages actually implemented in `memory.retrieve.v1`:

- identity, temporal, strategic, operational, analytics, episodic, conversational, procedural

These stages map to but don't exactly match the 10 memory types.

**7 Data Model Layers** (Neo4j Structure)
The physical organization of data in the graph database:

- Foundation, Vision, Strategy, Tactics, Execution, Track, Mind

These layers store the actual nodes and relationships that memory retrieval queries against.

### Memory-Related Terms

**Constitutional Memory**
The Foundation layer nodes (Principles, Values, Boundaries) that define ODEI's ethical framework and decision-making constraints. Immutable without explicit user consent.

**Memory Atlas**
The Neo4j graph database system that implements ODEI's persistent memory. Accessed via `odei-neo4j` MCP server.

**Semantic Search**
Vector-based similarity search across graph nodes, enabling contextually relevant memory retrieval without exact keyword matching.

**Health Dashboard**
Real-time monitoring interface (`odei-neo4j` health tool) showing memory system status, query performance, and database connectivity.

---

## Execution Concepts

### Subagents (Current)

Task-based autonomous behaviors defined in `.claude/agents/{name}.md`. Each file lists allowed MCP tools, token budgets,
and methodology. Agents invoke subagents via Claude Code’s **Task** tool, attach the file path, and provide a concise
brief. Tasks run in isolated mini-contexts, return compressed summaries, and keep heavy retrieval out of the main turn.

### Skills (Historical)

Legacy system where behaviors lived in `.claude/skills/{skill}/SKILL.md` and were manually loaded into the main context.
Skills were retired in favor of Task-based subagents; any references to `.claude/skills` now serve historical context
only.

---

## MCP Server Terminology

**MCP (Model Context Protocol)**
Anthropic's protocol for connecting Claude to external tools and data sources.

**odei-neo4j**
MCP server providing Memory Atlas access:

- Tools: `memory.retrieve.v1`, `memory.store.v1`, `memory.query.v1`, `health.v1`, `bootstrap.v1`
- Used by: All agents (discuss, plan, execute, mind)
- Built: TypeScript → `node dist/index.js`

**odei-history**
MCP server providing conversation persistence:

- Tools: Conversation storage and retrieval
- Used by: Root workspace (Electron UI), mind agent
- Built: TypeScript → `node dist/index.js`

**odei-miro**
MCP server providing Miro board integration:

- Tools: Board creation, item management, mind map visualization
- Used by: plan agent
- Built: TypeScript → `node dist/index.js`

**odei-apple**
MCP server providing macOS Calendar integration:

- Tools: Event management, calendar queries
- Used by: Root workspace (temporal context)
- Built: Swift → `swift run -c release`
- Platform-specific: Returns `UNSUPPORTED_PLATFORM` on non-macOS

---

## Workspace Concepts

### Workspace Types

**Root Workspace** (`/Users/ai/ODEI/`)
Primary workspace for Electron app development, UI work, and general project management.

- **MCP Servers:** `odei-neo4j`, `odei-history`, `odei-apple`
- **Use Cases:** Electron development, React/Preact UI, build configuration
- **Command:** `claude .` from `/Users/ai/ODEI/`

**Agent Workspaces** (`agents/*/`)
Individual workspaces for each autonomous agent with specialized MCP configurations.

- **Structure:** `agents/discuss/`, `agents/plan/`, `agents/execute/`, `agents/mind/`
- **MCP Config:** Each has `.claude/mcp.json` with agent-specific server configuration
- **Use Cases:** Agent-specific development and testing
- **Command:** `cd agents/<agent-name> && claude .`

### Workspace Configuration Files

**.claude/mcp.json**
Workspace-specific MCP server configuration. Defines which servers are available, how to start them, and environment variables.

⚠️ **Never use global `~/.claude/mcp.json`** - all configurations are workspace-specific.

**.claude/settings.local.json**
Workspace-specific Claude Code settings (not version controlled).

**.claude/CLAUDE.md**
Project-specific instructions for Claude Code (version controlled).

---

## Data Model Concepts

### Node Types

**Foundation Nodes**

- `Principle` - Core ethical principle (e.g., "Privacy by Design")
- `Value` - Personal value (e.g., "Transparency")
- `Boundary` - Hard constraint (e.g., "No data sharing without consent")

**Vision Nodes**

- `Goal` - Long-term objective (6+ months)
- `Aspiration` - Life direction or purpose

**Strategy Nodes**

- `Initiative` - Multi-month strategic project
- `Milestone` - Key strategic checkpoint

**Tactics Nodes**

- `Plan` - Week-to-month execution plan
- `Approach` - Specific tactical method

**Execution Nodes**

- `Task` - Concrete action item
- `Action` - Immediate step

**Track Nodes**

- `Metric` - Performance indicator
- `Measurement` - Specific metric value
- `Progress` - Status update

**Mind Nodes**

- `Insight` - Discovered pattern or learning
- `Retrospective` - Reflection on past events
- `Pattern` - Recurring theme or behavior

### Relationship Types

**ALIGNS_WITH** - Constitutional alignment
Connects execution-level nodes to Foundation layer principles, ensuring ethical consistency.

**SUPPORTS** - Hierarchical support
Links lower layers to higher layers (e.g., Task SUPPORTS Goal).

**DEPENDS_ON** - Dependency
Indicates execution dependencies (e.g., Task B DEPENDS_ON Task A).

**RELATES_TO** - Lateral connection
General association between nodes in same or different layers.

**MEASURES** - Metric tracking
Links Track layer metrics to execution or strategic nodes.

**DERIVES_FROM** - Learning lineage
Connects Mind layer insights to the observations that generated them.

---

## Common Confusions

### "Layer" Ambiguity

❌ **WRONG:** "ODEI has 3 layers"
✅ **RIGHT:** "ODEI has a 3-tier agent system, 7-layer data model, and 8-stage memory API"

**Guideline:** Always specify which architectural aspect you're referring to:

- Use "**tier**" for execution architecture (agent system)
- Use "**layer**" for data model (Neo4j graph)
- Use "**stage**" for memory retrieval (API)

### Memory Type Confusion

❌ **WRONG:** "There are 10 memory stages"
✅ **RIGHT:** "There are 10 conceptual memory types, but 8 retrieval stages in the API"

**Clarification:**

- **10 memory types** = design concept (episodic, semantic, procedural, etc.)
- **8 retrieval stages** = implementation (`memory.retrieve.v1` tool)
- **7 data model layers** = physical storage (Neo4j graph structure)

### Subagent Execution Confusion

❌ **WRONG:** "Subagents are separate processes running autonomously"
✅ **RIGHT:** "Subagents are currently Markdown methodology documents; future versions will support autonomous execution via Skills"

**Current State:** Subagents are `.md` files that agents read for guidance.
**Future State:** Subagents will be invokable via Skill tool for delegated execution.

### Agent Name Changes

❌ **WRONG:** "The decisions agent handles planning"
✅ **RIGHT:** "The plan agent (formerly called decisions) handles strategic planning"

**Historical Context:** Old documentation referred to the "decisions" agent. All current documentation should use "**plan**".

### Workspace Confusion

❌ **WRONG:** "Run `claude .` from root to work on the execute agent"
✅ **RIGHT:** "Run `cd agents/execute && claude .` to work on the execute agent with its specific MCP configuration"

**Rule:** Match your Claude Code workspace to your development context:

- Root workspace = Electron/UI work
- Agent workspace = Agent-specific development

### MCP Configuration Confusion

❌ **WRONG:** "Add MCP servers to `~/.claude/mcp.json` for global access"
✅ **RIGHT:** "Configure MCP servers in workspace-specific `.claude/mcp.json` files"

**Critical:** ODEI intentionally uses workspace-specific configurations. Never use global MCP config.

---

## Cross-Reference Index

### By Architectural Aspect

**Execution Architecture (3 Tiers)**

- See: 3-Tier Agent System, Agents, Subagents, Skills

**Data Model (7 Layers)**

- See: 7-Layer Data Model, Node Types, Relationship Types

**Memory System (8 Stages)**

- See: 8-Stage Memory API, Memory Architecture, Constitutional Memory

### By Agent

**discuss**

- Uses: odei-neo4j
- Related: Constitutional Memory, Foundation Layer, Semantic Search

**plan**

- Uses: odei-neo4j, odei-miro
- Related: Strategy Layer, Tactics Layer, Vision Layer
- Note: Formerly "decisions"

**execute**

- Uses: odei-neo4j
- Related: Execution Layer, Task Management, Operational Stage

**mind**

- Uses: odei-neo4j, odei-history
- Related: Mind Layer, Analytics Stage, Pattern Intelligence

### By MCP Server

**odei-neo4j**

- Implements: Memory Atlas, 7-Layer Data Model
- Used by: All agents
- Tools: memory.retrieve.v1, memory.store.v1, memory.query.v1, health.v1, bootstrap.v1

**odei-history**

- Implements: Conversation persistence
- Used by: Root workspace, mind agent
- Tools: Conversation storage/retrieval

**odei-miro**

- Implements: Mind map visualization
- Used by: plan agent
- Tools: Board/item management

**odei-apple**

- Implements: Calendar integration
- Used by: Root workspace
- Tools: Event management
- Note: macOS only

---

## Version History

- **2025-11-10:** Initial glossary creation
- Disambiguates 3-tier/7-layer/8-stage terminology
- Clarifies agent name changes (decisions → plan)
- Documents current vs. future subagent architecture

---

## Contributing

When adding new terminology:

1. Place definition in appropriate section
2. Add cross-references in index
3. Add common confusions if term is ambiguous
4. Update version history
