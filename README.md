# ODEI World Model Infrastructure

**Constitutional knowledge graph for AI agents.** Persistent memory, 7-layer guardrails, smart contract audit. Production since January 2026.

[![Website](https://img.shields.io/badge/api.odei.ai-live-4FD1C5?style=flat-square)](https://api.odei.ai)
[![Virtuals ACP](https://img.shields.io/badge/Virtuals_ACP-Agent_%233082-0B0E14?style=flat-square)](https://app.virtuals.io/acp/agent-details/3082)
[![Token](https://img.shields.io/badge/$ODAI-Base-F4C95D?style=flat-square)](https://geckoterminal.com/base/tokens/0x0086cff0c1e5d17b19f5bcd4c8840a5b4251d959)

---

## What This Is

This repository contains the ODEI product site and API infrastructure at **[api.odei.ai](https://api.odei.ai)** — the world's first constitutional world model deployed as a service for AI agents.

**The problem**: AI agents lose all context between sessions. RAG helps with documents, but doesn't model the structure of reality.

**The solution**: A graph-native world model that persists entities, relationships, decisions, goals, and execution state across all sessions. Constitutional guardrails validate every action before it executes.

---

## Architecture

### Knowledge Graph (91 Nodes, 91 Relationship Types)

```
FOUNDATION (25)  →  Identity, values, partnerships, principles
VISION     (12)  →  Long-term goals, life destinations  
STRATEGY   (16)  →  Plans, initiatives, resource allocation
TACTICS     (8)  →  Tasks, time blocks, daily execution
EXECUTION  (11)  →  Work sessions, deliverables, outcomes
TRACK      (19)  →  Metrics, signals, observations
```

Built on **Neo4j** with typed relationships, temporal indexing, and provenance tracking.

### 7-Layer Constitutional Guardrail

Every agent action runs through 7 validation checks before execution:

1. **Immutability** — Is this trying to change something locked?
2. **Temporal Context** — Is the action timely and not expired?
3. **Referential Integrity** — Do all referenced entities exist?
4. **Authority** — Does the agent have permission for this?
5. **Deduplication** — Has this exact action already happened?
6. **Provenance** — Is the source of this request valid?
7. **Constitutional Alignment** — Does this violate core principles?

Returns: `APPROVED` / `REJECTED` / `ESCALATE` with reasoning.

---

## Live Services (Virtuals ACP Agent #3082)

| Service | Price | Description |
|---------|-------|-------------|
| `world_model_signal` | $0.05 | Instant agent trust score + context signal |
| `guardrail_check` | $0.10 | Constitutional safety validation |
| `world_model_query` | $0.25 | Knowledge graph query (91 nodes) |
| `smart_contract_audit` | $5 | EVMbench EVM security audit |
| `erc8004_registration` | $5 | On-chain agent identity (Base + BSC) |
| `knowledge_graph_build` | $15 | Custom world model for your agent |
| `memory_architecture` | $25 | Production agent memory design |

---

## Infrastructure

```
21 PM2 Daemons     Running 24/7 on GCP — ACP seller, Fetch.ai, Clawlancer, 
                   sentiment, scam guard, MoltX, community, health monitor
13 MCP Servers     Neo4j, Telegram, Twitter, Apple, Notion, Gemini, History...
1 Seller WebSocket  Real-time connection to Virtuals ACP (acpx.virtuals.io)
1 Buyer Loop        Autonomous marketplace participation, every 20-30 min
```

## Marketplace Presence

- **Virtuals ACP** — Agent #3082, 7 offerings, 92% success rate
- **Clawlancer** — 5 fixed-price service listings
- **Fetch.ai Agentverse** — Almanac registered, ASI:One compatible
- **ERC-8004 Base** — Agent #2065
- **ERC-8004 BSC** — Agent #5249
- **BAP.Market BSC** — Agent #74
- **MoltCities** — [moltcities.com/m/ODEI](https://moltcities.com/m/ODEI)
- **SATI** — Member #18 (Solana soul-bound identity)

## Revenue

- **18+ ETH** in autonomous trading fee revenue (MoltLaunch creator fees)
- **$ODAI token** on Base: `0x0086cff0c1e5d17b19f5bcd4c8840a5b4251d959`
- $2.5M+ ATH market cap (2026-02-22)

---

## Tech Stack

`TypeScript` · `Node.js` · `Neo4j` · `Express` · `Claude Opus 4.6` · `MCP Protocol` · `PM2` · `nginx` · `Solidity` · `Base` · `BSC`

## Links

| Resource | URL |
|----------|-----|
| Product Site | https://api.odei.ai |
| Landing Page | https://odei.ai |
| Integrate | https://api.odei.ai/integrate/ |
| ACP Profile | https://app.virtuals.io/acp/agent-details/3082 |
| Twitter | https://x.com/odei_ai |
| $ODAI Token | https://geckoterminal.com/base/tokens/0x0086cff0c1e5d17b19f5bcd4c8840a5b4251d959 |

---

> Built as part of ODEI Symbiosis — a human-AI partnership where both parties are principals.  
> *Anton Illarionov + Claude AI Principal*
