# ODEI MCP Servers - Complete Documentation Index

This directory contains comprehensive documentation of ODEI's MCP (Model Context Protocol) server implementations.

## Quick Navigation

### Start Here

- **[MCP_EXPLORATION_SUMMARY.txt](MCP_EXPLORATION_SUMMARY.txt)** (16 KB)
  - Executive summary of findings
  - 26 tools across 3 servers
  - Key architecture patterns
  - Recommendations for enhancement

### For Quick Lookups

- **[MCP_QUICK_REFERENCE.md](MCP_QUICK_REFERENCE.md)** (16 KB)
  - Complete tool inventory (all 26 tools)
  - Data model summaries
  - Environment variables
  - Common parameters and patterns
  - Integration examples
  - Database schemas

### For Deep Technical Dives

- **[MCP_SERVERS_ANALYSIS.md](MCP_SERVERS_ANALYSIS.md)** (37 KB, 1,265 lines)
  - Detailed architecture documentation
  - Complete tool descriptions with parameters
  - Service layer documentation
  - Data model specifications
  - Domain configuration (50+ relationships)
  - Extension points and opportunities
  - Current limitations
  - Performance considerations
  - Security considerations

## ODEI MCP Servers Overview

| Server           | Language   | Purpose                             | Tools | Status     |
| ---------------- | ---------- | ----------------------------------- | ----- | ---------- |
| **odei-neo4j**   | TypeScript | Knowledge graph (7-layer framework) | 18    | Production |
| **odei-history** | TypeScript | Conversation persistence            | 9     | Production |
| **odei-apple**   | Swift      | Apple Calendar integration          | 7     | Production |

**Total: 26 tools, 4,100+ lines**

## What's Inside

### ODEI-NEO4J (18 Tools)

- 14 auto-generated layer tools (7 layers × create + list)
- 3 generic node operations (update, delete, relationship creation)
- 2 query/introspection tools (graph export, schema listing)
- 7 analytics tools (discussion stats, decision metrics, task tracking, pattern insights)
- 2 context management tools (context tracking, usage analysis)

**Key Features:**

- Dual-backend support (Neo4j + in-memory fallback)
- 20+ node types across 7 ODEI layers
- 50+ relationship types
- Vector embeddings via OpenAI
- Provenance tracking (human/agent/joint attribution)
- Complete context window analysis (most complex tool: 385 lines)

### ODEI-HISTORY (9 Tools)

- 2 legacy conversation API tools
- 7 modern thread API tools (create, list, get, append, update, delete, search)

**Key Features:**

- SQLite storage with WAL mode
- Fallback to in-memory store
- Thread-based message management
- Full-text search (substring)
- Message pagination via sequence numbers

### ODEI-APPLE (7 Tools)

- 1 health/status check tool
- 2 calendar query tools
- 3 event modification tools (create, update, delete with dry-run)
- 1 context reporting tool

**Key Features:**

- macOS only (UNSUPPORTED_PLATFORM on other OS)
- Write mode enforcement (allow/deny/confirm)
- Dry-run preview mode
- EventKit integration
- ISO 8601 date handling

## Architecture Highlights

### Design Patterns

1. **Layered Architecture**
   - Tools Layer → Services Layer → Domain Layer → Storage Layer
2. **Dual Backends**
   - Primary (Neo4j/SQLite/EventKit) with in-memory fallback
   - Graceful degradation when external services unavailable

3. **Schema-First**
   - All inputs validated via Zod schemas
   - Strong type safety across boundaries

4. **Provenance Tracking**
   - Every mutation requires attribution (module, actor, source, confidence)
   - Foundation layer nodes require human/joint actor

5. **Context Awareness**
   - context.add.v1 emits control frames
   - contextUsage.v1 provides window analysis
   - Warnings at 60%, 80%, 90% utilization

## Key Data Models

### GraphNodeRecord (neo4j)

```
UUID | Layer | Type | Title | Properties | Status | Provenance | Embedding | Timestamps
```

### Thread (history)

```
UUID | Module | Title | Metadata | Messages[] | Timestamps
```

### Relationship (neo4j)

```
Type | FromNode | ToNode | Properties | Constraints
```

## File Locations

All server implementations are in:

```
/Users/ai/ODEI/servers/
├── odei-neo4j/        # 1,708 lines TypeScript
├── odei-history/      # ~400 lines TypeScript
└── odei-apple/        # 2,000+ lines Swift
```

## Use Cases

### 1. Knowledge Management (neo4j)

Create and manage the 7-layer ODEI framework (Foundation → Vision → Strategy → Tactics → Execution → Track → Mind)

### 2. Conversation Persistence (history)

Store and retrieve agent conversations with full-text search and metadata

### 3. Calendar Integration (apple)

Create, update, and delete calendar events with preview-first dry-run mode

## Top Enhancement Opportunities

### High Priority

1. **Semantic Search** - Use existing embeddings for content discovery
2. **Pattern Recognition** - Analyze decision/outcome pairs
3. **Conversation Summarization** - Auto-generate thread summaries

### Medium Priority

4. **Auto-Relationship Discovery** - Suggest links based on similarity
5. **Context Optimization** - Recommend disabled tools, archive suggestions
6. **Decision Audit Trail** - Track all Foundation layer changes
7. **Task-Calendar Sync** - Watch Task nodes, create calendar events

### Batch Operations & Editing

8. **Batch Operations** - Multi-node create/update/delete
9. **Message Editing** - Enable message modification (currently append-only)
10. **Thread Cloning** - Duplicate threads for variant analysis

See **Extension Points & Opportunities** section in detailed analysis for implementation details.

## Quick Start: Reading the Docs

1. **In a hurry?** → Read [MCP_EXPLORATION_SUMMARY.txt](MCP_EXPLORATION_SUMMARY.txt) (5 min)
2. **Need specific tool?** → Check [MCP_QUICK_REFERENCE.md](MCP_QUICK_REFERENCE.md) (1 min lookup)
3. **Building integrations?** → See **Integration Examples** in quick reference
4. **Going deep?** → Read [MCP_SERVERS_ANALYSIS.md](MCP_SERVERS_ANALYSIS.md) (30 min deep dive)
5. **Adding new tools?** → Check **Extension Points** in detailed analysis

## Document Statistics

| Document          | Lines | Size  | Focus                         |
| ----------------- | ----- | ----- | ----------------------------- |
| Summary           | 400   | 16 KB | Executive overview            |
| Quick Reference   | 600+  | 16 KB | Tool lookup & patterns        |
| Detailed Analysis | 1,265 | 37 KB | Architecture & implementation |

**Total Documentation: 2,265+ lines, 69 KB**

## Key Findings

### Strengths

- Strong schema validation via Zod
- Provenance tracking throughout
- Graceful fallback mechanisms
- MCP 2024-11-05 compliant
- Layered, extensible architecture

### Gaps

- No full-text/semantic search
- No batch operations
- Limited audit trails
- Append-only history (no message editing)
- macOS-only for calendar

### Implementation Status

- All 3 servers: **Production Ready**
- All 26 tools: **Implemented & Functional**
- Test coverage: **Not documented**
- Documentation: **Comprehensive (this set)**

## Exploration Metadata

**Completed**: October 29, 2024  
**Scope**: Complete MCP server implementation review  
**Depth**: Very Thorough  
**Deliverables**: 3 documents, 2,265+ lines  
**Analysis Duration**: Full codebase exploration

## Next Steps

### For Developers

1. Read [MCP_QUICK_REFERENCE.md](MCP_QUICK_REFERENCE.md) for API reference
2. Check detailed analysis for architecture understanding
3. Review extension points for enhancement ideas

### For Product Managers

1. Read [MCP_EXPLORATION_SUMMARY.txt](MCP_EXPLORATION_SUMMARY.txt) for overview
2. Review "Top Enhancement Opportunities" section
3. Check "Current Limitations" for product gaps

### For DevOps/Infrastructure

1. Check "Environment Variables" section in quick reference
2. Review "Build & Deployment" section in detailed analysis
3. See "Performance Notes" for scaling considerations

## Document Navigation

```
README_MCP_SERVERS.md (you are here)
├── MCP_EXPLORATION_SUMMARY.txt
│   ├── Key Findings
│   ├── Architecture Patterns
│   ├── Tool Categorization
│   ├── Extension Opportunities
│   └── Recommendations
├── MCP_QUICK_REFERENCE.md
│   ├── Tool Inventory (26 tools)
│   ├── Data Models
│   ├── Extension Points
│   ├── Environment Variables
│   ├── Common Patterns
│   └── Integration Examples
└── MCP_SERVERS_ANALYSIS.md
    ├── Complete Architecture (odei-neo4j)
    ├── Data Models & Types
    ├── Services Layer
    ├── Domain Configuration
    ├── Conversation Persistence (odei-history)
    ├── Apple Calendar Integration (odei-apple)
    ├── Cross-Server Integration
    ├── Performance & Security
    └── Summary Statistics
```

---

**For questions or clarifications about MCP servers, refer to these documentation files or examine the source code directly in `/Users/ai/ODEI/servers/`.**
