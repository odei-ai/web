# ODEI MCP Servers API Documentation

Comprehensive API reference for all ODEI Model Context Protocol (MCP) servers.

## Quick Reference

| Server | Tools | Description |
|--------|-------|-------------|
| [odei-neo4j](./odei-neo4j.md) | 28 | Memory Atlas — Constitutional graph, Vision/Strategy/Tactics layers, search, relationships |
| [odei-history](./odei-history.md) | 10 | Conversation History — Thread management, message storage, search across conversations |
| [odei-telegram](./odei-telegram.md) | 3 | Telegram Integration — Chat management, message sending/receiving, notifications |
| [odei-conductor](./odei-conductor.md) | 5 | Agent Coordination — Multi-agent task dispatch, status tracking, inter-agent communication |
| [odei-gemini](./odei-gemini.md) | 2 | Context Search — Semantic search across ODEI knowledge, preset context retrieval |
| [odei-health](./odei-health.md) | 2 | Health Intelligence — Readiness scores, biometric tracking, capacity recommendations |
| [odei-apple](./odei-apple.md) | 0 | Apple Health Integration — HealthKit data access, vitals, workouts, sleep tracking |

## Server Details

### odei-neo4j

Memory Atlas — Constitutional graph, Vision/Strategy/Tactics layers, search, relationships

**Tools:** 28

[View full documentation →](./odei-neo4j.md)

- `odei.neo4j.backup.restore.v1` — No description provided
- `odei.neo4j.cache.management.v1` — No description provided
- `odei.neo4j.context.add.v1` — No description provided
- `odei.neo4j.context.usage.v1` — No description provided
- `odei.neo4j.discuss.activeThreads.v1` — No description provided
- ... and 23 more

### odei-history

Conversation History — Thread management, message storage, search across conversations

**Tools:** 10

[View full documentation →](./odei-history.md)

- `odei.history.conversations.create.v1` — No description provided
- `odei.history.conversations.query.v1` — No description provided
- `odei.history.messages.recent.v1` — No description provided
- `odei.history.threads.appendMessage.v1` — No description provided
- `odei.history.threads.create.v1` — No description provided
- ... and 5 more

### odei-telegram

Telegram Integration — Chat management, message sending/receiving, notifications

**Tools:** 3

[View full documentation →](./odei-telegram.md)

- `odei.telegram.chats.list` — No description provided
- `odei.telegram.messages.fetch` — No description provided
- `odei.telegram.messages.send` — No description provided

### odei-conductor

Agent Coordination — Multi-agent task dispatch, status tracking, inter-agent communication

**Tools:** 5

[View full documentation →](./odei-conductor.md)

- `odei.conductor.cancelTask.v1` — Cancel a running task. Use this to stop a task that is taking too long or is no longer needed.
- `odei.conductor.dispatch.v1` — Dispatch a task to another agent. Orchestration clients use this to assign work.
- `odei.conductor.getOutput.v1` — Get output from an agent or task.
- `odei.conductor.status.v1` — Get status of all agents and tasks. Use this to check which agents are running and their current tasks.
- `odei.conductor.taskComplete.v1` — Mark a task as completed. Agents call this when they finish their assigned work.

### odei-gemini

Context Search — Semantic search across ODEI knowledge, preset context retrieval

**Tools:** 2

[View full documentation →](./odei-gemini.md)

- `odei.gemini.context.preset.v1` — No description provided
- `odei.gemini.context.search.v1` — No description provided

### odei-health

Health Intelligence — Readiness scores, biometric tracking, capacity recommendations

**Tools:** 2

[View full documentation →](./odei-health.md)

- `odei.health.readiness.v1` — Get detailed readiness analysis with factors, alerts, and recommendations.
- `odei.health.state.v1` — Get current health state summary from Apple Watch and Garmin data.

### odei-apple

Apple Health Integration — HealthKit data access, vitals, workouts, sleep tracking

**Tools:** 0

[View full documentation →](./odei-apple.md)


## Usage

### In Agent Workspaces

Each agent workspace configures the MCP servers it needs in `.claude/mcp.json`:

```json
{
  "mcpServers": {
    "odei-neo4j": {
      "command": "node",
      "args": ["/Users/ai/ODEI/servers/odei-neo4j/dist/index.js"],
      "env": {
        "NEO4J_URI": "bolt://localhost:7687",
        "NEO4J_USERNAME": "neo4j",
        "NEO4J_PASSWORD": "your-password"
      }
    }
  }
}
```

### Building Servers

TypeScript servers must be built before use:

```bash
cd servers/odei-neo4j
npm install
npm run build
```

Swift servers (odei-apple):

```bash
cd servers/odei-apple
swift build -c release
```

## Documentation Updates

This documentation is auto-generated from tool definitions:

```bash
npm run docs:generate
```

Documentation is regenerated automatically during builds to ensure accuracy.
