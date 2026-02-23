# Agent Memory: Why RAG Isn't Enough

**The case for graph-native persistent memory in AI agents.**

## The Problem

Every AI agent has the same fundamental issue: when the session ends, everything the agent learned disappears.

Context windows are caches — fast and flexible, but volatile. Vector RAG retrieves documents well but doesn't model **relationships** between entities, track **decisions** across sessions, or enforce **constitutional constraints** on what the agent can do.

## What Agents Actually Need

Agents need to remember:
- That they **already tried** X and it didn't work
- That entity A **blocked** entity B three weeks ago
- That this decision **relates to** a goal from two months ago
- That this action **violates** a constitutional rule

These are **graph problems**, not document problems.

## ODEI's Graph-Native Approach

```
Session 1: Agent creates task "Deploy guardrails" → stored as TACTICS node
Session 2: Agent queries TACTICS → finds existing task, doesn't recreate
Session 3: Agent checks EXECUTION → sees deployment was completed, skips
```

No hallucination. No duplication. Constitutional validation at every step.

## Architecture

ODEI uses Neo4j with 91 nodes across 6 semantic domains:

```cypher
(Agent)-[:PURSUES_GOAL]->(Goal)
(Goal)-[:ACHIEVED_BY]->(Task)
(Task)-[:BLOCKED_BY]->(Risk)
(Decision)-[:INFORMED_BY]->(Evidence)
(Metric)-[:TRACKS]->(Goal)
```

91 relationship types. Fully traversable. Temporal indexing for "what was true when."

## Integration

```bash
# MCP server (Claude Desktop)
npx @odei/mcp-server

# Direct API
curl -H "Authorization: Bearer TOKEN" \
  https://api.odei.ai/api/v2/world-model/live
```

## Available as a Service

ODEI's world model is available via:
- **Virtuals ACP**: $0.25/query at [Agent #3082](https://app.virtuals.io/acp/agent-details/3082)
- **REST API**: api.odei.ai/api/v2/
- **MCP**: github.com/odei-ai/mcp-odei
- **Fetch.ai**: ASI:One marketplace

## Further Reading

- [Architecture Overview](architecture.md)
- [Constitutional Guardrails](guardrails.md)
- [Integration Guide](https://api.odei.ai/integrate/)
