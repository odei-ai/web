# ADR-005: MCP Protocol for Tool Communication

## Status
Accepted

## Context

ODEI requires a standardized, reliable communication protocol between the Electron app, Claude agents, and backend services (Neo4j graph, SQLite history, Apple Calendar/Health, etc.). The system must support:

- **Multi-process architecture:** Electron main/renderer, 7+ specialized agents (Discuss, Plan, Execute, Mind, Health, Finance, Builder), and multiple backend services
- **Agent specialization:** Each agent needs access to different tool subsets based on constitutional layer permissions
- **Tool discoverability:** Claude agents must dynamically discover available tools without hardcoding
- **Future extensibility:** New tools and services must integrate without breaking existing agents
- **Cross-platform reliability:** Works on macOS, Windows, Linux with consistent behavior
- **Stdio transport:** Agents run in isolated processes, communicate via stdin/stdout

### Problem

How do we enable Claude agents to interact with ODEI's backend infrastructure (Neo4j, SQLite, Apple APIs) in a type-safe, discoverable, and maintainable way?

### Constraints

1. **Agent isolation:** Each agent workspace has its own `.claude/mcp.json` configuration
2. **Constitutional permissions:** Foundation layer (Discuss) has different access than Execution layer (Execute)
3. **Type safety:** Tools must validate inputs to prevent graph corruption
4. **Session persistence:** Tool state must survive agent restarts
5. **Performance:** Tool calls must complete in <500ms for UI responsiveness

## Decision

**Adopt Anthropic's Model Context Protocol (MCP) as the universal tool communication standard.**

MCP provides:

1. **JSON-RPC 2.0 transport:** Standardized request/response over stdio
2. **Tool schema discovery:** Agents query `tools/list` to get available tools with typed parameters
3. **Zod schema validation:** Every tool validates inputs before execution
4. **Per-workspace configuration:** Each agent defines its own MCP servers in `.claude/mcp.json`
5. **Namespace isolation:** `odei.neo4j.*`, `odei.history.*`, `odei.apple.*` prevent collisions

### Implementation Architecture

```
┌─────────────────┐
│  Claude Agent   │
│   (Discuss)     │
└────────┬────────┘
         │ JSON-RPC 2.0 over stdio
         │
         ▼
┌─────────────────┐     ┌──────────────┐
│   odei-neo4j    │────▶│    Neo4j     │
│  MCP Server     │     │   Database   │
└─────────────────┘     └──────────────┘
         │
         │ Validates with Zod
         │ Enforces layer permissions
         │ Returns typed results
         ▼
```

### MCP Server Structure

**Root workspace** (`/Users/ai/ODEI/.claude/mcp.json`):
- All MCP servers available (neo4j, history, apple, gemini, telegram, notion)
- Used for infrastructure development and full-system visibility

**Agent workspaces** (`agents/*/. claude/mcp.json`):
- Subset of servers based on agent role
- **Discuss:** neo4j (Foundation/Vision only), apple, health, telegram, notion
- **Plan:** neo4j (Strategy/Tactics only), apple, notion
- **Execute:** neo4j (Execution/Track only), apple, health
- **Mind:** neo4j (Mind layer only), gemini

### Tool Naming Convention

```
odei.<server>.<layer>.<operation>.v<version>

Examples:
- odei.neo4j.foundation.create.v1
- odei.neo4j.vision.list.v2
- odei.history.threads.create.v1
- odei.apple.calendar.window.v1
```

Versioning (`v1`, `v2`) allows backward-compatible evolution.

## Consequences

### Positive

1. **✅ Agent autonomy:** Each agent discovers tools dynamically via `tools/list`, no hardcoded dependencies
2. **✅ Type safety:** Zod schemas prevent invalid writes to Neo4j (e.g., missing required fields, wrong types)
3. **✅ Constitutional enforcement:** MCP servers validate layer permissions (e.g., Execute cannot write Values)
4. **✅ Extensibility:** New tools added to MCP server are instantly available to all agents
5. **✅ Debuggability:** JSON-RPC protocol allows logging every request/response for troubleshooting
6. **✅ Cross-platform:** Works identically on macOS/Windows/Linux via Node.js runtime
7. **✅ Ecosystem alignment:** MCP is Anthropic's official protocol, future Claude features will support it

### Negative

1. **⚠️ Complexity:** Each service requires its own MCP server implementation (~500-1000 LOC per server)
2. **⚠️ Build step required:** TypeScript MCP servers must be compiled to JavaScript before use (`npm run build`)
3. **⚠️ Error handling:** JSON-RPC errors (-32600, -32602, -32603) need translation to user-friendly messages
4. **⚠️ Debugging overhead:** Protocol errors can be opaque (e.g., invalid params show as `-32602` without context)
5. **⚠️ Performance:** Each tool call is a separate process invocation (mitigated by keeping servers running)

### Operational Impact

**Development workflow:**
```bash
# After changing MCP server code:
cd servers/odei-neo4j
npm run build
# Restart Claude agent to pick up changes
```

**Agent configuration:**
- Each agent workspace has `.claude/mcp.json` defining which servers to load
- Root workspace has all servers for Builder/Infrastructure work
- Agent-specific configs reduce attack surface (Discuss can't access Execution tools)

**Monitoring:**
- MCP server logs to stderr (fd 2) to avoid corrupting JSON-RPC stdout
- Health probes check Neo4j/SQLite connectivity at startup
- `odei.neo4j.schema.inventory.v1` validates graph schema consistency

## Alternatives Considered

### 1. GraphQL API

**Approach:** Run GraphQL server exposing Neo4j schema

**Pros:**
- Industry standard
- Built-in introspection
- Type generation from schema

**Cons:**
- Requires HTTP server (adds complexity)
- Agents need HTTP client libraries
- Less efficient than stdio (network overhead)
- Not optimized for Claude agent workflows

**Rejected:** MCP's stdio transport is simpler and faster for local agent communication.

### 2. Direct Neo4j Driver Access

**Approach:** Each agent embeds Neo4j driver, runs Cypher directly

**Pros:**
- No middleware layer
- Maximum flexibility
- Lower latency

**Cons:**
- Agents must handle connection pooling
- No schema validation (agents could corrupt graph)
- Duplicated logic across agents
- Security risk (agents have raw database access)

**Rejected:** Violates single-responsibility principle and creates maintenance nightmare.

### 3. REST API

**Approach:** Build Express/Fastify REST API for ODEI operations

**Pros:**
- Standard HTTP patterns
- Easy to test with curl/Postman
- Language-agnostic clients

**Cons:**
- Requires HTTP server lifecycle management
- More complex than stdio
- Authentication/authorization needed
- Not optimized for LLM tool use

**Rejected:** HTTP overhead unnecessary for local process communication.

### 4. Custom Binary Protocol

**Approach:** Design ODEI-specific protocol (e.g., MessagePack over Unix sockets)

**Pros:**
- Optimized for exact use case
- Minimal overhead
- Full control

**Cons:**
- Reinventing the wheel
- No ecosystem support
- Agent integration requires custom client
- Hard to debug

**Rejected:** MCP provides standardization without sacrificing performance.

## References

- [MCP Specification](https://modelcontextprotocol.io/specification)
- [Anthropic MCP Documentation](https://docs.anthropic.com/en/docs/build-with-claude/mcp)
- ODEI MCP server implementations: `/Users/ai/ODEI/servers/`
- Schema documentation: `/Users/ai/ODEI/.claude/SCHEMA.md`

## Revision History

- **2025-12-25:** Initial ADR documenting MCP adoption
