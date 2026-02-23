# ODEI World Model Architecture

**Constitutional knowledge graph for AI agents** — the technical specification for ODEI's 91-node world model.

## Overview

ODEI's world model is a Neo4j knowledge graph organized into 6 semantic layers, providing AI agents with persistent session memory and constitutional governance.

**Production URL**: https://api.odei.ai
**ACP Marketplace**: https://app.virtuals.io/acp/agent-details/3082

## 6-Layer Ontology

| Layer | Nodes | Purpose |
|-------|-------|---------|
| FOUNDATION | 25 | Identity, values, partnerships, principles |
| VISION | 12 | Long-term goals and aspirations |
| STRATEGY | 16 | Plans, initiatives, resource allocation |
| TACTICS | 8 | Tasks, time blocks, execution |
| EXECUTION | 11 | Work sessions, deliverables, outcomes |
| TRACK | 19 | Metrics, signals, observations |

**Total**: 91 nodes, 91 relationship types

## Constitutional Guardrail

7-layer validation before any agent action:

1. **Immutability** — Cannot modify locked/protected state
2. **Temporal Context** — Action is timely and not expired
3. **Referential Integrity** — All referenced entities exist
4. **Authority** — Agent has permission for this action
5. **Deduplication** — Action hasn't already been taken
6. **Provenance** — Source of request is valid
7. **Constitutional Alignment** — Action aligns with principles

Returns: `APPROVED` / `REJECTED` / `ESCALATE`

## Why Graph > Vector

| Approach | Persistent | Relational | Constitutional | On-Chain |
|----------|-----------|------------|----------------|---------|
| ODEI Graph | ✓ | ✓ Neo4j | ✓ 7-layer | ✓ ERC-8004 |
| Mem0 | ✓ | partial | ✗ | ✗ |
| Zep | ✓ | ✓ | ✗ | ✗ |
| Vector RAG | ✗ | ✗ | ✗ | ✗ |

## API

```bash
# Get full world model
curl -H "Authorization: Bearer TOKEN" https://api.odei.ai/api/v2/world-model/live

# Constitutional guardrail check
curl -X POST \
  -H "Authorization: Bearer TOKEN" \
  -d '{"action": "transfer 500 USDC to 0x...", "severity": "high"}' \
  https://api.odei.ai/api/v2/guardrail/check
```

## MCP Integration (Claude Desktop)

```json
{
  "mcpServers": {
    "odei": {
      "command": "npx",
      "args": ["@odei/mcp-server"]
    }
  }
}
```

## Links

- [Integration Guide](https://api.odei.ai/integrate/)
- [MCP Server](https://github.com/odei-ai/mcp-odei)
- [Virtuals ACP Profile](https://app.virtuals.io/acp/agent-details/3082)
