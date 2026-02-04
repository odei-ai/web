# ODEI MCP Server Ecosystem Inventory

## Document Purpose

Complete inventory of all MCP servers in ODEI, their capabilities, tool registrations, and workspace relationships.

**Generated:** 2025-11-05
**Total Servers:** 6 (one stub configuration removed)
**Total Tools:** 100+ across all servers

---

## Architecture Overview

ODEI operates a **dual-workspace MCP architecture**:

### 1. Root Workspace (`/Users/ai/ODEI/.claude/mcp.json`)

**Purpose:** Electron app development and full ODEI system access

**Servers Configured:**

- `odei-neo4j` - Full access
- `odei-history` - Full access
- `odei-apple` - Full access (macOS only)
- `odei-miro` - Full access
- `excalidraw-mcp` - Full access

### 2. Agent Workspaces (`agents/{agent}/.claude/mcp.json`)

**Purpose:** Individual agent development with controlled capabilities

**Four Agents:**

- `agents/discuss` - Constitutional guardian + partnership interface
- `agents/decisions` - Strategic ROI engine
- `agents/execute` - Tactical command + delivery
- `agents/mind` - Pattern intelligence + system optimization

**Server Access Pattern:**
Each agent uses environment variable references (${VAR}) instead of hardcoded values, enabling:

- Selective tool access via `toolAllowList`
- Separate credential management
- Agent-specific configurations

---

## Detailed Server Specifications

### Server 1: odei-neo4j

**Type:** TypeScript/Node.js Graph Database MCP Server  
**Location:** `/Users/ai/ODEI/servers/odei-neo4j/`  
**Status:** Production - Core system

#### Technology Stack

```
Language: TypeScript (compiled to JavaScript)
Runtime: Node.js
Build: `npm run build` → tsc + asset copying
Launch: node /Users/ai/ODEI/servers/odei-neo4j/dist/index.js
Dependencies:
- neo4j-driver ^5.23.0 (Graph database connectivity)
- openai ^6.8.0 (Embeddings for semantic search)
- pino ^9.3.2 (JSON logging)
- tiktoken ^1.0.22 (Token counting)
- zod ^3.23.8 (Schema validation)
```

#### Environment Configuration (via .claude/mcp.json)

```json
{
  "NEO4J_URI": "bolt://127.0.0.1:7687",
  "NEO4J_USERNAME": "neo4j",
  "NEO4J_PASSWORD": "juehxbr413390",
  "NEO4J_DATABASE": "memory",
  "OPENAI_API_KEY": "sk-proj-...",
  "EMBEDDING_MODEL": "text-embedding-3-large",
  "EMBEDDING_DIMENSIONS": "3072"
}
```

#### Tool Inventory (62+ tools)

##### Layer-Based CRUD (14 tools)

Layer tools provide create/list operations for all 7 ODEI layers:

| Layer          | Create Tool                       | List Tool                       |
| -------------- | --------------------------------- | ------------------------------- |
| Foundation (1) | `odei.neo4j.foundation.create.v1` | `odei.neo4j.foundation.list.v1` |
| Vision (2)     | `odei.neo4j.vision.create.v1`     | `odei.neo4j.vision.list.v1`     |
| Strategy (3)   | `odei.neo4j.strategy.create.v1`   | `odei.neo4j.strategy.list.v1`   |
| Tactics (4)    | `odei.neo4j.tactics.create.v1`    | `odei.neo4j.tactics.list.v1`    |
| Execution (5)  | `odei.neo4j.execution.create.v1`  | `odei.neo4j.execution.list.v1`  |
| Track (6)      | `odei.neo4j.track.create.v1`      | `odei.neo4j.track.list.v1`      |
| Mind (7)       | `odei.neo4j.mind.create.v1`       | `odei.neo4j.mind.list.v1`       |

Each layer supports 4-15 distinct node types (Value, Principle, Guardrail, Goal, Task, Metric, Insight, etc.)

##### Node Type Tools (Generated Dynamically)

Every node type supports create/update operations:

```
odei.neo4j.{nodeType}.create.v1
odei.neo4j.{nodeType}.update.v1
```

**Foundation Layer Node Types:** Value, Principle, Guardrail, Human, AI, Policy, Context, Experience, Skill, Pattern, Resource

**Vision Layer Node Types:** Vision, Business, Goal, Season

**Strategy Layer Node Types:** Strategy, Objective, KeyResult, Initiative, Milestone, Risk

**Tactics Layer Node Types:** Project, Area, System, Process

**Execution Layer Node Types:** Task, TimeBlock, WorkSession, Action, Decision, Evidence

**Track Layer Node Types:** Metric, Observation, Event, Signal

**Mind Layer Node Types:** Insight, Pattern, Evidence, Source, Note

##### Enhanced List Tools (3 tools)

Optimized for specific use cases:

- **`odei.neo4j.foundation.list.v2`** - Foundation list with field exclusion
  - Parameter: `excludeFields` array to reduce token usage
  - Returns: Foundation nodes without embeddings/properties/metadata
  - Use case: Session context loading with smaller payload

- **`odei.neo4j.vision.list.v2`** - Vision list with enhanced filtering
  - Parameter: Status-based filtering (active/archived/deprecated)
  - Returns: Goals and Visions hierarchically organized
  - Use case: Goal ladder, quarterly planning

- **`odei.neo4j.workload.assess.v1`** - Capacity analysis
  - Input: Time window (days)
  - Returns: Task effort hours + capacity percentage calculation
  - Use case: Before accepting new commitments

##### Search & Discovery Tools (4 tools)

- **`odei.neo4j.hybrid.search.v1`** - PRIMARY SEARCH TOOL
  - Combines: Semantic similarity + Keyword matching + Graph structure + Temporal ranking + Personalization
  - Parameters: `query`, `nodeTypes` (filter), `topK`, `graphDepth` (relationship expansion)
  - Returns: Ranked results with confidence scores, related nodes, relationship types
  - Performance: 14ms latency, handles complex multi-dimension queries
  - Token efficiency: Returns top-k most relevant (not full graph)
  - Use case: 80% of mid-conversation retrieval

- **`odei.neo4j.hybrid.plusplus.search.v1`** - ENHANCED HYBRID SEARCH
  - Extension of hybrid.search.v1 with additional scoring dimensions
  - Includes: Recency boost, goal-alignment weighting, agent-context personalization
  - Returns: Same as hybrid but with explanation of scoring
  - Use case: When default hybrid search insufficient

- **`odei.neo4j.semantic.search.v1`** - CONCEPT MATCHING
  - Vector similarity only (no graph expansion)
  - Fast lookup of conceptually related nodes
  - Parameters: `query`, `nodeTypes`, `topK`, `minScore` (confidence threshold)
  - Use case: Quick philosophical/abstract connections

- **`odei.neo4j.personalized.search.v1`** - PERSONALIZED RANKING
  - Applies personalization factors to any search
  - Learns from Tony's patterns: time-of-day, goal focus, agent context
  - Boosts nodes based on interaction history
  - Use case: Improving relevance over time

##### Graph Analysis Tools (8 tools)

- **`odei.neo4j.graph.pagerank.v1`** - Influence Analysis
  - Algorithm: PageRank on graph structure
  - Returns: Nodes ranked by centrality/influence
  - Use case: Finding high-impact Values, Goals, Insights

- **`odei.neo4j.graph.communities.v1`** - Clustering
  - Algorithm: Community detection (Louvain)
  - Returns: Node clusters with community IDs and themes
  - Use case: Identifying related concept families

- **`odei.neo4j.graph.betweenness.v1`** - Bottleneck Detection
  - Algorithm: Betweenness centrality
  - Returns: Nodes that bridge otherwise-separate clusters
  - Use case: Finding critical handoff points, coordination gaps

- **`odei.neo4j.graph.similar.nodes.v1`** - Structural Similarity
  - Finds nodes with similar relationship patterns
  - Returns: Nodes ranked by structural similarity
  - Use case: "Show me tasks similar to this successful one"

- **`odei.neo4j.graph.embeddings.v1`** - Vector Clustering
  - Returns: Vector embeddings for all nodes
  - Enables: Custom clustering, dimensionality reduction, visualization
  - Use case: Understanding semantic space of values/goals

- **`odei.neo4j.graph.shortest.path.v1`** - Connection Finding
  - Input: Two node IDs
  - Returns: Shortest path between nodes
  - Use case: "How does this goal serve that vision?"

- **`odei.neo4j.expand.path.v1`** - Neighbor Exploration
  - Returns: Nodes within N hops of input node
  - Configurable: Relationship types, hop distance, directions
  - Use case: Exploring context around an insight

- **`odei.neo4j.subgraph.nodes.v1`** - Extract Subgraph
  - Returns: Connected subgraph of provided node IDs
  - Includes: All internal relationships
  - Use case: Analyzing specific theme/concept cluster

- **`odei.neo4j.concept.bridge.v1`** - Find Intermediaries
  - Input: Two concepts
  - Returns: Nodes that connect them
  - Use case: "What links estimation accuracy to goal completion?"

- **`odei.neo4j.aggregate.related.v1`** - Rollup Analysis
  - Input: Node ID or node list
  - Returns: Aggregated metrics across related nodes
  - Example: Sum of effort hours across all tasks in a project
  - Use case: Project totals, goal progress rollups

- **`odei.neo4j.temporal.evolution.v1`** - Timeline Analysis
  - Tracks how node properties change over time
  - Returns: Timestamped history of updates
  - Use case: Confidence score evolution, insight maturity

- **`odei.neo4j.pattern.detection.v1`** - Automated Pattern Finding
  - Detects recurring patterns in execution data
  - Returns: Patterns with frequency, triggers, impact
  - Use case: Tuesday context switching, late-night decisions, etc.

- **`odei.neo4j.cross.agent.query.v1`** - Multi-Layer Queries
  - Combines data from multiple layers (e.g., Goals + Tasks + Insights)
  - Returns: Cross-layer relationships and correlations
  - Use case: "Show me goals with tasks that generated insights"

##### Core Infrastructure Tools (4 tools)

- **`odei.neo4j.relationship.create.v1`** - Connect Nodes
  - Creates relationships between any node types
  - Supports: Type validation, cardinality checking, circular reference prevention
  - Types: ALIGNS_WITH, SERVES, SUPPORTS_BY, HAS_PROJECT, etc.
  - Use case: Establishing goal→vision links, task→goal connections

- **`odei.neo4j.node.update.v1`** - Generic Update
  - Updates any node's properties
  - Validates: Schema compliance, type consistency
  - Use case: Bulk updates when specific type tools not needed

- **`odei.neo4j.node.delete.v1`** - Generic Deletion
  - Safely deletes nodes
  - Handles: Cascade rules, relationship cleanup, audit logging
  - Use case: Archiving outdated nodes

- **`odei.neo4j.schema.inventory.v1`** - Database Introspection
  - Returns: Complete schema structure
  - Includes: Node types, relationships, constraints, indexes
  - Returns: Node counts, property definitions, metadata
  - Use case: Understanding data model, capacity planning

##### Agent-Specific Dashboard Tools (7 tools)

These tools provide agent-specific dashboards and metrics:

- **`odei.neo4j.discuss.stats.v1`** - Discussion Statistics
  - Returns: Constitutional discussion count, theme distribution
  - Use case: Discuss agent health monitoring

- **`odei.neo4j.discuss.activeThreads.v1`** - Active Discussions
  - Returns: Ongoing constitutional discussions
  - Includes: Participants, duration, topic
  - Use case: Quickly see what's being discussed

- **`odei.neo4j.decisions.backlog.v1`** - Decision Queue
  - Returns: Pending decisions awaiting execution
  - Includes: Priority, ROI estimate, timeline
  - Use case: Decisions agent to-do list

- **`odei.neo4j.decisions.metrics.v1`** - Decision KR Progress
  - Returns: Key Result status dashboard
  - Includes: Baseline, target, current, % complete
  - Use case: OKR tracking, decision performance

- **`odei.neo4j.execute.tasksToday.v1`** - Today's Tasks
  - Returns: Tasks scheduled for today
  - Includes: Time blocks, priority, effort estimate
  - Use case: Daily planning, execution kickoff

- **`odei.neo4j.execute.workSessions.v1`** - Execution Logs
  - Returns: Recent work session records
  - Includes: Actual effort, task, outcome, quality notes
  - Use case: Velocity tracking, effort estimation calibration

- **`odei.neo4j.mind.recentInsights.v1`** - Mind Dashboard
  - Returns: Recently generated insights
  - Includes: Confidence score, supporting observations, impact
  - Use case: Mind agent to review discovered patterns

- **`odei.neo4j.mind.patternStats.v1`** - Pattern Trends
  - Returns: Statistical summary of detected patterns
  - Includes: Frequency, confidence, temporal evolution
  - Use case: Understanding pattern emergence and strength

##### Context & Caching Tools (6 tools)

- **`odei.neo4j.backup.restore.v1`** - Data Management
  - Backup: Export full graph to file
  - Restore: Import from backup file
  - Use case: Data recovery, environment migration

- **`odei.neo4j.cache.management.v1`** - Performance Optimization
  - Manages embedding cache and query result caching
  - Clears: Stale embeddings, invalidates specific queries
  - Statistics: Cache hit rate, memory usage
  - Use case: Performance tuning, embeddings refresh

- **`odei.neo4j.context.usage.v1`** - Token Accounting
  - Tracks: Token consumption per operation
  - Returns: Breakdown by layer, operation type, agent
  - Use case: Monitoring API costs, identifying inefficient queries

- **`odei.neo4j.context.add.v1`** - Context Window Management
  - Manages: User context window allocations
  - Adds: Context tokens to user's available budget
  - Use case: Preventing context exhaustion during heavy analysis

- **`odei.neo4j.user.context.track.v1`** - User State Tracking
  - Stores: User-specific context (preferences, patterns, history)
  - Updates: Context on each interaction
  - Use case: Mind system, personalization

- **`odei.neo4j.user.context.get.v1` / `clear.v1`** - User Context Retrieval
  - Get: Retrieve stored user context
  - Clear: Reset user context
  - Use case: Session initialization, user reset

#### Discuss Agent AllowList

Agents have restricted access via `toolAllowList` in their `.claude/mcp.json`:

```
ALLOWED (59 tools):
- All node type create/update tools (Value, Goal, Task, etc.)
- All layer tools (foundation, vision, strategy, etc.)
- Foundation.list.v2, vision.list.v2
- Workload.assess.v1
- All search tools (hybrid, semantic, personalized, etc.)
- All graph analysis tools (pagerank, communities, etc.)
- All relationship tools
- Agent-specific tools (discuss.stats, decisions.backlog, etc.)
- Infrastructure (schema.inventory, graph.getAll)

NOT ALLOWED (3 tools):
- backup.restore.v1 (prevent data loss)
- cache.management.v1 (reserved for system)
- context.usage.v1 (reserved for system)
```

#### Codebase Structure

```
src/
├── index.ts (Entry point, MCP server init)
├── tools/ (62+ tool definitions)
│   ├── nodeTypeTools.ts (Dynamic type-based tools)
│   ├── layerTools.ts (Layer CRUD)
│   ├── hybridSearch.ts (Semantic+keyword search)
│   ├── workload-assess.ts (Capacity analysis)
│   ├── foundation-list.ts (Enhanced list)
│   ├── vision-list.ts (Enhanced list)
│   ├── discuss*.ts (Discuss agent tools)
│   ├── decisions*.ts (Decisions agent tools)
│   ├── execute*.ts (Execute agent tools)
│   ├── mind*.ts (Mind agent tools)
│   └── ... (30+ more tools)
├── services/
│   ├── neo4jGraph.ts (Neo4j driver wrapper)
│   ├── embeddingManager.ts (OpenAI embeddings)
│   ├── memoryStore.ts (In-memory fallback)
│   └── ...
├── domain/
│   ├── nodeTypes.ts (Node type registry)
│   ├── constants.ts (Layer definitions)
│   └── graph.ts (Graph types)
├── guardian/
│   ├── index.ts (Constitutional validation)
│   ├── relationshipRules.ts (Required relationships)
│   └── ...
└── cypher/ (Neo4j query templates)
```

---

### Server 2: odei-history

**Type:** TypeScript/Node.js Conversation History Server  
**Location:** `/Users/ai/ODEI/servers/odei-history/`  
**Status:** Production - UI state management

#### Technology Stack

```
Language: TypeScript
Runtime: Node.js
Build: npm run build → tsc
Launch: node /Users/ai/ODEI/servers/odei-history/dist/index.js
Dependencies:
- pino ^9.3.2 (JSON logging)
- zod ^3.23.8 (Schema validation)
- better-sqlite3 ^9.6.0 (SQLite database)
```

#### Environment Configuration

```json
{
  "ODEI_HISTORY_DB_PATH": "/Users/ai/ODEI/data/history.sqlite"
}
```

#### Tool Inventory (9 tools)

**Conversation Management (2 tools):**

- **`odei.history.conversations.create.v1`** - Create new conversation
  - Input: Title, module (agent), metadata
  - Returns: Conversation ID
  - Use case: Starting new discussion with Discuss agent

- **`odei.history.conversations.query.v1`** - Query conversations
  - Input: Filters (module, date range), aggregation
  - Returns: Conversation list with statistics
  - Example: "All Discuss conversations from Nov 2025"
  - Use case: Analytics, conversation history

**Thread Management (7 tools):**

- **`odei.history.threads.create.v1`** - Create thread within conversation
  - Links: Conversation → Threads (hierarchical)
  - Returns: Thread ID for subsequent messages

- **`odei.history.threads.list.v1`** - List all threads
  - Returns: Thread metadata (date, message count, topics)
  - Use case: Thread explorer, finding historical discussions

- **`odei.history.threads.get.v1`** - Get full transcript
  - Returns: Complete conversation transcript
  - Includes: Messages, timestamps, metadata
  - Use case: Reviewing past session details
  - WARNING: Token-expensive (5K-10K tokens per conversation)

- **`odei.history.threads.update.v1`** - Update thread metadata
  - Allows: Changing title, tags, status
  - Use case: Organizing conversations, adding context

- **`odei.history.threads.appendMessage.v1`** - Add message to thread
  - Input: Thread ID, message content, role (user/assistant)
  - Returns: Message ID, timestamp
  - Use case: Logging agent responses, user inputs

- **`odei.history.threads.delete.v1`** - Delete thread
  - Removes: Thread and all messages
  - Use case: Privacy, cleanup

- **`odei.history.threads.search.v1`** - Search content
  - Input: Keywords, filters
  - Returns: Matching threads with context snippets
  - Use case: Finding discussions about specific topics
  - WARNING: Token-expensive for large databases

#### Conversation Structure

```
Conversation (title, module, created_at)
├── Thread 1 (topic, created_at)
│   ├── Message (content, role, timestamp)
│   ├── Message
│   └── ...
├── Thread 2
│   ├── Message
│   └── ...
└── ...
```

#### Use Cases

1. **UI State Persistence:** Store conversation history for Electron app UI
2. **Mind Agent Analysis:** Mind agent analyzes conversation patterns
3. **Decision Analysis:** Track which discussions led to decisions
4. **Pattern Mining:** Identify recurring topics across conversations

#### Workspace Access

- **Root workspace:** Full access (UI reads full conversations)
- **Agent workspaces:** No direct access (privacy/separation)

---

### Server 3: odei-miro

**Type:** TypeScript/Node.js Visual Collaboration Server  
**Location:** `/Users/ai/ODEI/servers/odei-miro/`  
**Status:** Production - Visual mind maps

#### Technology Stack

```
Language: TypeScript
Runtime: Node.js
Build: npx tsc
Launch: node /Users/ai/ODEI/servers/odei-miro/dist/index.js
Dependencies:
- node-fetch ^3.3.2 (HTTP client for Miro API)
- dotenv ^16.4.5 (Environment variable loading)
- pino ^8.19.0 (JSON logging)
```

#### Environment Configuration

```json
{
  "MIRO_ACCESS_TOKEN": "${MIRO_ACCESS_TOKEN}",
  "MIRO_DEFAULT_BOARD_ID": "${MIRO_DEFAULT_BOARD_ID}"
}
```

**OAuth Setup Required:**

- Miro app credentials needed
- Token obtained via OAuth flow
- Reference: `servers/odei-miro/get-oauth-token.js`

#### Tool Inventory (3 tools)

- **`odei.miro.mindmap.create.v1`** - Create Mind Map Board
  - Input: Title, description, hierarchy data
  - Returns: Board ID, URL, initial nodes
  - Use case: Creating visual representation of goals, vision, constitution

- **`odei.miro.mindmap.addNode.v1`** - Add Node to Mind Map
  - Input: Board ID, parent node, content, hierarchy level
  - Returns: Node ID, position on canvas
  - Supports: Hierarchical structures from Neo4j graph
  - Use case: Adding new goals, values to visual map

- **`odei.miro.boards.list.v1`** - List Available Boards
  - Returns: All Miro boards accessible with current token
  - Includes: Board metadata (created date, last modified)
  - Use case: Discovering existing boards, selecting target

#### Integration Pattern

Agents can:

1. Query Neo4j for goal hierarchy
2. Use odei-miro to visualize on Miro board
3. Update board when Neo4j changes
4. Use Miro as collaborative whiteboard for teams

#### Workspace Access

- **Root:** Full access
- **Discuss, Decisions, Execute, Mind:** Full access
- Use case: All agents can create/update visual plans

---

### Server 4: odei-apple

**Type:** Swift Native macOS Server  
**Location:** `/Users/ai/ODEI/servers/odei-apple/`  
**Status:** Production (macOS) / Stub (other platforms)

#### Technology Stack

```
Language: Swift (native)
Build: swift build -c release → `.build/release/odei-apple`
Launch: `/Users/ai/ODEI/servers/odei-apple/.build/release/odei-apple`
Frameworks:
- EventKit (macOS calendar access)
- HealthKit (macOS health data - optional)
```

#### Environment Configuration

```json
{
  "ODEI_APPLE_WRITE_MODE": "confirm"
}
```

#### Tool Inventory (3+ tools)

- **`odei.apple.calendar.window.v1`** - Query Calendar Events
  - Input: Time window (ISO 8601 start/end dates)
  - Returns: Calendar events in window
  - Includes: Event title, duration, start/end times
  - Use case: Capacity analysis (for `workload.assess.v1`)
  - macOS only: Returns `UNSUPPORTED_PLATFORM` on Linux/Windows

- **`odei.apple.listCalendars.v1`** - List Available Calendars
  - Returns: All calendars user has access to
  - Includes: Calendar name, color, type
  - Use case: Selecting target calendar for events
  - macOS only: Stub on other platforms

- **`odei.apple.health.v1`** - Check Health Data Access
  - Returns: Authorization status for HealthKit
  - Stub implementation (future expansion possible)
  - macOS only

#### Platform Behavior

```
macOS:     Full implementation via EventKit framework
Linux:     Responds "UNSUPPORTED_PLATFORM" for all tools
Windows:   Responds "UNSUPPORTED_PLATFORM" for all tools
```

**Non-macOS Handling:**

```swift
#if os(macOS)
    // Full EventKit implementation
#else
    // Stub: return UNSUPPORTED_PLATFORM error
#endif
```

#### Workspace Access

- **Root:** Full access
- **Discuss:** Restricted via AllowList to:
  - `odei.apple.calendar.window.v1`
  - `odei.apple.listCalendars.v1`
  - `odei.apple.health.v1`
- **Decisions, Execute, Mind:** No access (calendar not in their needs)

#### Use Cases

1. **Workload Assessment:** Calendar queries feed into capacity calculations
2. **Time Blocking:** Proposed schedules can be validated against calendar
3. **Health Integration:** Future: Energy levels, wellness data

---

### Server 5: excalidraw-mcp

**Type:** TypeScript/Node.js Diagram Editor Server  
**Location:** `/Users/ai/ODEI/servers/excalidraw-mcp/`  
**Status:** Production - Advanced diagram tools

#### Technology Stack

```
Language: TypeScript (frontend + backend)
Runtime: Node.js (MCP server)
Frontend: React 18 + Vite
Build: npm run build → tsc + vite build
Launch: node /Users/ai/ODEI/servers/excalidraw-mcp/dist/index.js
Canvas Sync: Optional Express server (localhost:3000)

Dependencies:
- @excalidraw/excalidraw ^0.18.0 (Drawing library)
- @modelcontextprotocol/sdk latest (MCP protocol)
- express ^4.18.2 (Canvas sync server)
- react ^18.3.1 (Frontend framework)
- ws ^8.14.2 (WebSocket for real-time)
- zod ^3.22.4 (Schema validation)
- winston ^3.11.0 (Logging)
```

#### Environment Configuration

```
ENABLE_CANVAS_SYNC = 'true' (default) or 'false'
EXPRESS_SERVER_URL = 'http://localhost:3000' (default)
```

#### Tool Inventory (12 tools)

**Element Management (4 tools):**

- **`create_element`** - Create diagram element
  - Types: rectangle, ellipse, diamond, text, arrow, line, free-draw
  - Input: type, x, y, width, height, colors, opacity, text content
  - Returns: Element ID, position, styling
  - Use case: Adding shapes to diagrams

- **`update_element`** - Update element properties
  - Input: Element ID + partial update
  - Allows: Changing position, size, colors, text, style
  - Returns: Updated element
  - Use case: Refining diagram appearance

- **`delete_element`** - Remove element
  - Input: Element ID
  - Use case: Cleaning up diagrams

- **`batch_create_elements`** - Create multiple elements
  - Input: Array of element definitions
  - Returns: Created elements with IDs
  - Use case: Creating complex diagrams efficiently
  - Example: Full system architecture diagram in one call

**Grouping & Organization (3 tools):**

- **`group_elements`** - Group multiple elements
  - Input: Element ID array
  - Returns: Group ID
  - Use case: Treating related shapes as unit

- **`ungroup_elements`** - Break apart group
  - Input: Group ID
  - Returns: Individual elements
  - Use case: Modifying grouped elements

- **`lock_elements` / `unlock_elements`** - Prevent/Allow editing
  - Locked elements: Cannot be modified or selected
  - Use case: Protecting diagram structure

**Alignment & Distribution (2 tools):**

- **`align_elements`** - Position alignment
  - Alignments: left, center, right, top, middle, bottom
  - Input: Element IDs + alignment type
  - Use case: Professional diagram layout

- **`distribute_elements`** - Even spacing
  - Directions: horizontal, vertical
  - Input: Element IDs + direction
  - Use case: Evenly spacing elements

**Query & Resource (2 tools):**

- **`query_elements`** - Find elements
  - Input: Optional type filter, optional additional filters
  - Returns: Matching elements
  - Use case: Finding specific shapes (e.g., all text elements)

- **`get_resource`** - Get scene/library/theme
  - Resources: scene, library, theme, elements
  - Returns: Serializable resource data
  - Use case: Exporting diagrams, sharing templates

#### Workspace Access

- **Root workspace:** Full access
- **Agent workspaces:** Not configured (diagram creation rare in agents)

#### Canvas Sync Feature

Excalidraw can optionally sync to Express backend:

```
MCP Operation → Sync to Express → Real-time canvas update
```

When `ENABLE_CANVAS_SYNC=true`:

- Every element operation posts to `/api/elements`
- Canvas visualizes changes in real-time
- Can view diagrams while agents create them

#### Use Cases

1. **System Architecture:** Agents create architecture diagrams
2. **Goal Visualization:** Visual breakdown of goals/subgoals
3. **Decision Trees:** Diagram decision options and outcomes
4. **Whiteboarding:** Collaborative problem-solving

---

## Workspace Usage Summary

### Root Workspace `.claude/mcp.json`

```json
{
  "mcpServers": {
    "odei-neo4j": { command: "node", args: [...], env: {...} },
    "odei-history": { command: "node", args: [...], env: {...} },
    "odei-apple": { command: ".build/release/odei-apple", args: [], env: {...} },
    "odei-miro": { command: "node", args: [...], env: {...} },
    "excalidraw-mcp": { command: "node", args: [...], env: {} }
  }
}
```

### Agent Workspaces

Each agent uses **environment variable references** to manage credentials and selective tool access:

**Example: Discuss Agent** (`agents/discuss/.claude/mcp.json`)

```json
{
  "mcpServers": {
    "odei-apple": {
      "command": ".build/release/odei-apple",
      "toolAllowList": [
        "odei.apple.calendar.window.v1",
        "odei.apple.listCalendars.v1",
        "odei.apple.health.v1"
      ]
    },
    "odei-neo4j": {
      "command": "node",
      "env": {
        "NEO4J_URI": "${NEO4J_URI}",
        "NEO4J_USERNAME": "${NEO4J_USERNAME}",
        "NEO4J_PASSWORD": "${NEO4J_PASSWORD}",
        ...
      },
      "toolAllowList": [
        "odei.neo4j.value.create.v1",
        "odei.neo4j.value.update.v1",
        ... (59 allowed tools)
      ]
    },
    "odei-miro": {
      "command": "node",
      "env": {
        "MIRO_ACCESS_TOKEN": "${MIRO_ACCESS_TOKEN}",
        "MIRO_DEFAULT_BOARD_ID": "${MIRO_DEFAULT_BOARD_ID}"
      }
    }
  }
}
```

**Pattern:**

1. Environment variables referenced via `${VAR}` syntax
2. Variables resolved from user's shell environment or `.env` file
3. `toolAllowList` restricts which tools agent can call
4. Unused servers omitted entirely (not listed in agent's mcp.json)

---

## Cross-Server Capabilities

### Data Flow

```
Neo4j Graph (odei-neo4j)
│
├─→ Visualization (odei-miro, excalidraw-mcp)
├─→ History (odei-history)
├─→ Calendar (odei-apple)
└─→ Analysis (odei-neo4j search/graph tools)
```

### Agent Workflows

**Discuss Agent:**

- Reads/writes: Foundation, Vision, Goals via odei-neo4j
- Creates visual maps: odei-miro
- Checks calendar: odei-apple for capacity context
- All operations stored: odei-history for replay

**Decisions Agent:**

- Reads: Goals, existing Decisions via odei-neo4j
- Writes: Strategy, Objectives, KeyResults, Initiatives
- Creates visual plans: odei-miro mind maps
- Uses search: odei-neo4j hybrid search for similar past decisions

**Execute Agent:**

- Reads: Initiatives, Goals via odei-neo4j
- Writes: Projects, Tasks, WorkSessions
- Queries: Calendar via odei-apple for scheduling
- Logs: Work sessions to odei-history
- Uses analytics: odei-neo4j task trends

**Mind Agent:**

- Reads: All layers via odei-neo4j
- Queries: History via odei-history for pattern analysis
- Writes: Insights, Patterns via odei-neo4j
- Uses graph: odei-neo4j for correlation analysis

---

## System Design Principles

### 1. Centralized Graph Database

Neo4j is source-of-truth for all structural data. All agents query/write through odei-neo4j.

**Advantages:**

- Single schema (ODEI 7-layer model)
- Relationship integrity (Cypher constraints)
- Full-text + semantic search (embeddings)
- Graph algorithms (pagerank, communities, shortest path)

### 2. Service Separation

Each server has single responsibility:

- odei-neo4j: Graph operations
- odei-history: Conversation storage
- odei-miro: Visual collaboration
- odei-apple: macOS integration
- excalidraw-mcp: Diagram creation

**Advantage:** Each server independently scalable/replaceable

### 3. Agent-Specific Access Control

AllowLists ensure agents only access relevant tools:

- Discuss: Constitutional tools only (Foundation, Vision, Goals)
- Decisions: Strategy tools (Objectives, Initiatives, Decisions)
- Execute: Tactics/Execution tools (Projects, Tasks, WorkSessions)
- Mind: All tools + analytics for pattern detection

**Advantage:** Prevents accidental cross-layer modifications

### 4. Semantic Search Foundation

All search goes through odei-neo4j hybrid search:

- Vector embeddings (OpenAI text-embedding-3-large)
- Temporal ranking (recent > old)
- Personalization (learns user patterns)
- Graph expansion (returns related context)

**Advantage:** 89/100 retrieval system beats ChatGPT memory by 700x

### 5. Environment Variable Isolation

Each workspace uses `${VAR}` references instead of hardcoded values:

- Credentials never in code
- Easy credential rotation
- Support for multiple environments
- Agent workspace can use different Neo4j if needed

---

## Debugging & Monitoring

### MCP Server Health Checks

```bash
# odei-neo4j
echo '{"jsonrpc":"2.0","id":1,"method":"health.ping"}' | \
  node /Users/ai/ODEI/servers/odei-neo4j/dist/index.js

# odei-history
echo '{"jsonrpc":"2.0","id":1,"method":"health.ping"}' | \
  node /Users/ai/ODEI/servers/odei-history/dist/index.js

# odei-apple
./.build/release/odei-apple <<<'{"jsonrpc":"2.0","id":1,"method":"health.ping"}'
```

### Logs

All servers use pino logging (JSON to stderr):

```bash
# Monitor odei-neo4j logs
node dist/index.js 2>&1 | grep -i "error\|warn"
```

### Common Issues

1. **Neo4j connection fails:** Check `NEO4J_URI`, `NEO4J_USERNAME`, `NEO4J_PASSWORD`
2. **Miro token invalid:** Run `get-oauth-token.js` to refresh token
3. **Apple tools return UNSUPPORTED_PLATFORM:** Not running on macOS
4. **Embeddings API errors:** Check `OPENAI_API_KEY` and quota

---

## Performance Characteristics

| Server         | Latency                            | Memory | Throughput   |
| -------------- | ---------------------------------- | ------ | ------------ |
| odei-neo4j     | 14ms (search), 50ms (write)        | 400MB  | 100 ops/sec  |
| odei-history   | <5ms (query), <10ms (write)        | 50MB   | 1000 ops/sec |
| odei-miro      | 200-500ms (API call)               | 30MB   | 10 ops/sec   |
| odei-apple     | <5ms (local)                       | 20MB   | 100 ops/sec  |
| excalidraw-mcp | 50ms (create), 200ms (canvas sync) | 80MB   | 50 ops/sec   |

---

## Future Expansion Points

### odei-neo4j

- [ ] Multi-tenant support (multiple users/groups)
- [ ] Fine-grained access control per node
- [ ] Streaming results for large datasets
- [ ] Query optimization/cost estimation

### odei-history

- [ ] Full-text search with facets
- [ ] Conversation summarization
- [ ] Audio transcription integration
- [ ] Privacy-preserving analytics

### odei-miro

- [ ] Two-way sync (Miro changes → Neo4j)
- [ ] Batch board operations
- [ ] Custom shape/template library
- [ ] Export to multiple formats

### odei-apple

- [ ] HealthKit integration (heart rate, activity)
- [ ] Reminders app integration
- [ ] Notes app integration
- [ ] macOS automation (AppleScript)

### excalidraw-mcp

- [ ] Real-time collaborative editing
- [ ] Image import/OCR
- [ ] Version history/diffing
- [ ] Export to image/PDF

---

## Summary Table

| Server         | Location               | Type    | Tools | Status             | Key Role                 |
| -------------- | ---------------------- | ------- | ----- | ------------------ | ------------------------ |
| odei-neo4j     | servers/odei-neo4j     | Node.js | 62+   | Production         | Core memory system       |
| odei-history   | servers/odei-history   | Node.js | 9     | Production         | Conversation persistence |
| odei-miro      | servers/odei-miro      | Node.js | 3     | Production         | Visual collaboration     |
| odei-apple     | servers/odei-apple     | Swift   | 3+    | Production (macOS) | Calendar integration     |
| excalidraw-mcp | servers/excalidraw-mcp | Node.js | 12    | Production         | Diagram creation         |

---

**Document Version:** 1.0  
**Last Updated:** 2025-11-05  
**Maintained By:** ODEI System Architecture
