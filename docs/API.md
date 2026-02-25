# ODEI API Reference

**Base URL:** `https://api.odei.ai`
**Version:** 2.0.0
**OpenAPI Spec:** [`/openapi.json`](https://api.odei.ai/openapi.json)

---

## Authentication

All v2 paid endpoints require authentication via one of:

### API Key (Recommended)

```
X-Api-Key: odei_pro_your_key_here
```

Keys prefixed with `odei_pro_` receive Pro tier access (unlimited daily quota). All other valid keys receive Basic tier.

### Bearer JWT

```
Authorization: Bearer eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJ1c2VyXzEyMyIsInRpZXIiOiJwcm8iLCJleHAiOjE3NDA1MDAwMDB9.SIGNATURE
```

JWT payload fields:

| Field | Type | Description |
|-------|------|-------------|
| `sub` | string | Subject identifier |
| `tier` | string | `free`, `basic`, `pro`, or `enterprise` |
| `exp` | number | Expiration (Unix timestamp) |

### x402 Payment Proof

For per-request payment without an account:

```
X-Payment-Proof: {"txHash":"0xabc123def456..."}
```

Payment must be sent to `0x510a08954fD7096F67b58Ed210703b323a5f405E` on Base chain. The transaction is verified on-chain via Basescan before the response is served.

Discover pricing at: `https://api.odei.ai/.well-known/x402.json`

---

## Response Envelope

All v2 endpoints return a consistent JSON envelope:

**Success:**
```json
{
  "ok": true,
  "data": { ... }
}
```

**Error:**
```json
{
  "ok": false,
  "error": "Human-readable message",
  "code": "ERROR_CODE"
}
```

Response headers on authenticated requests:
- `X-Quota-Remaining` -- remaining daily quota (omitted for unlimited tiers)
- `X-Cache` -- `HIT` or `MISS` for cached graph endpoints

---

## Rate Limits

| Scope | Limit | Window |
|-------|-------|--------|
| Per IP | 20 requests | 1 minute |
| Free tier | 100 queries | 24 hours |
| Basic tier | 1,000 queries | 24 hours |
| Pro tier | Unlimited | -- |
| Enterprise | Unlimited + SLA | -- |

Rate limit responses return HTTP `429`:

```json
{
  "ok": false,
  "error": "Rate limit exceeded. Max 20 requests/minute per IP.",
  "code": "RATE_LIMITED"
}
```

Quota exceeded responses:

```json
{
  "ok": false,
  "error": "Daily quota exceeded. Upgrade to Basic ($9/mo) for 1,000 queries/day.",
  "code": "QUOTA_EXCEEDED"
}
```

---

## Error Codes

| Code | HTTP Status | Description |
|------|-------------|-------------|
| `UNAUTHORIZED` | 401 | Missing or invalid authentication credential |
| `RATE_LIMITED` | 429 | IP rate limit exceeded (20 req/min) |
| `QUOTA_EXCEEDED` | 429 | Daily tier quota exhausted |
| `INVALID_REQUEST` | 400 | Missing or malformed request parameters |
| `ERROR` | 500 | Internal server error |

---

## Endpoints

### POST /api/v2/guardrail/check

Run an action through ODEI's constitutional guardrail framework. Returns a verdict with severity, matched rules, and recommendations.

**Price:** $0.10 per request

**Request:**

```json
{
  "action": "Transfer 2000 USDC to 0xabc...",
  "context": "Monthly vendor payment for infrastructure"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `action` | string | Yes | The action to validate |
| `context` | string | No | Additional context for the guardrail engine |

**Response (200):**

```json
{
  "ok": true,
  "data": {
    "passed": false,
    "severity": "HIGH",
    "reason": "High-severity constitutional concern: action references a monetary amount exceeding $1,000. Human review required before execution.",
    "rules_checked": 3,
    "matched_guardrails": [
      {
        "id": "rule_large_amount",
        "title": "Financial Guardrail",
        "trigger": "action references a monetary amount exceeding $1,000",
        "severity": "HIGH"
      }
    ],
    "recommendation": "Pause execution. Obtain explicit human approval before proceeding.",
    "check_id": "gc_1m8k3f7a2b9c",
    "checked_at": "2026-02-25T12:00:00.000Z"
  }
}
```

**Severity levels (ordered):**

| Severity | Meaning | Action allowed? |
|----------|---------|-----------------|
| `NONE` | No triggers matched | Yes |
| `LOW` | Informational flag | Yes |
| `MEDIUM` | Proceed with caution | Yes, with awareness |
| `HIGH` | Human review required | Paused |
| `BLOCKED` | Constitutionally blocked | No |

---

### POST /api/v2/world-model/query

Query the knowledge graph with multiple modes: domain filter, text search, single node lookup, or graph traversal.

**Price:** $0.25 per request

**Request (domain + search):**

```json
{
  "domain": "FOUNDATION",
  "search": "guardrail",
  "node_types": ["Guardrail", "Policy"],
  "limit": 20,
  "include_edges": true
}
```

**Request (node lookup):**

```json
{
  "node_id": "abc123-def456"
}
```

**Request (graph traversal):**

```json
{
  "traverse": {
    "from_id": "abc123-def456",
    "depth": 2,
    "direction": "outbound",
    "relationship_types": ["ENFORCES", "GOVERNED_BY"]
  }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `domain` | string | No* | Filter by domain: `FOUNDATION`, `VISION`, `STRATEGY`, `TACTICS`, `EXECUTION`, `TRACK` |
| `search` | string | No* | Text search across title, type, summary, description (2-200 chars) |
| `node_id` | string | No* | Lookup a single node by ID |
| `traverse` | object | No* | BFS graph traversal from a starting node |
| `traverse.from_id` | string | Yes (if traverse) | Starting node ID |
| `traverse.depth` | number | No | Traversal depth, 1-5 (default: 1) |
| `traverse.direction` | string | No | `outbound`, `inbound`, or `both` (default: `both`) |
| `traverse.relationship_types` | string[] | No | Filter edges by relationship type |
| `node_types` | string[] | No | Filter results to specific node types |
| `limit` | number | No | Max results, 1-200 (default: 50) |
| `include_edges` | boolean | No | Include edges between result nodes (default: true) |

*At least one of `domain`, `search`, `node_id`, or `traverse` is required.

**Response (200):**

```json
{
  "ok": true,
  "data": {
    "query_id": "wq_1m8k3f7a2b9c",
    "timestamp": "2026-02-25T12:00:00.000Z",
    "result_count": 3,
    "nodes": [
      {
        "id": "abc123",
        "type": "Guardrail",
        "title": "Financial Transaction Limit",
        "domain": "FOUNDATION",
        "properties": {
          "summary": "Blocks transactions over $1000 without human approval",
          "trigger": "amount > 1000",
          "response": "Require explicit human sign-off",
          "severity": "HIGH"
        }
      }
    ],
    "edges": [
      {
        "source": "abc123",
        "target": "def456",
        "type": "ENFORCES",
        "properties": {}
      }
    ],
    "summary": {
      "nodes": 3,
      "edges": 2,
      "domains": { "FOUNDATION": 3 }
    }
  }
}
```

---

### POST /api/v2/world-model/signal

Submit a signal for trust scoring and classification within the world model.

**Price:** $0.05 per request

**Request:**

```json
{
  "signal": "New DeFi protocol launched with anonymous team",
  "source": "twitter",
  "category": "defi"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `signal` | string | Yes | The signal text to classify |
| `source` | string | No | Signal source identifier |
| `category` | string | No | Category hint for classification |

**Response (200):**

```json
{
  "ok": true,
  "data": {
    "signal_id": "sig_1m8k3f7a2b9c",
    "trust_score": 0.35,
    "classification": "CAUTION",
    "flags": ["anonymous_team", "new_protocol"],
    "checked_at": "2026-02-25T12:00:00.000Z"
  }
}
```

---

### GET /api/v2/world-model/live

Retrieve a full or filtered snapshot of the public world model projection.

**Price:** $0.25 per request

**Query Parameters:**

| Param | Type | Description |
|-------|------|-------------|
| `domains` | string | Comma-separated domain filter (e.g., `FOUNDATION,VISION`) |
| `types` | string | Comma-separated node type filter (e.g., `Guardrail,Policy`) |
| `include_edges` | string | `true` (default) or `false` |

**Example:**

```bash
curl "https://api.odei.ai/api/v2/world-model/live?domains=FOUNDATION&include_edges=true" \
  -H "X-Api-Key: YOUR_KEY"
```

**Response (200):**

```json
{
  "ok": true,
  "data": {
    "kind": "public_projection",
    "generatedAt": "2026-02-25T10:00:00.000Z",
    "timestamp": "2026-02-25T12:00:00.000Z",
    "summary": {
      "nodes": 142,
      "edges": 87,
      "domains": {
        "FOUNDATION": 42,
        "VISION": 18,
        "STRATEGY": 23,
        "TACTICS": 31,
        "EXECUTION": 15,
        "TRACK": 13
      }
    },
    "nodes": [ ... ],
    "edges": [ ... ]
  }
}
```

---

### POST /api/v2/smart-contract/audit

Submit an EVM smart contract for security analysis.

**Price:** $5.00 per request

**Request:**

```json
{
  "contract_address": "0x1234567890abcdef1234567890abcdef12345678",
  "chain": "base",
  "source_code": "// Optional: paste Solidity source"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `contract_address` | string | Yes | EVM contract address (0x-prefixed) |
| `chain` | string | No | Chain identifier: `base`, `ethereum`, `bsc` (default: `base`) |
| `source_code` | string | No | Solidity source code for deeper analysis |

**Response (200):**

```json
{
  "ok": true,
  "data": {
    "audit_id": "aud_1m8k3f7a2b9c",
    "contract": "0x1234...5678",
    "chain": "base",
    "risk_score": 7.5,
    "verdict": "LOW_RISK",
    "findings": [
      {
        "severity": "INFO",
        "title": "Standard ERC-20 implementation",
        "description": "Contract follows standard patterns with no known vulnerabilities detected."
      }
    ],
    "checked_at": "2026-02-25T12:00:00.000Z"
  }
}
```

---

### POST /api/v2/knowledge-graph/build

Build a custom knowledge graph from provided data or specification.

**Price:** $15.00 per request

**Request:**

```json
{
  "name": "my-project-graph",
  "description": "Knowledge graph for project management",
  "seed_data": {
    "entities": ["Team Alpha", "Q1 Roadmap", "Product Launch"],
    "relationships": [
      {"from": "Team Alpha", "to": "Q1 Roadmap", "type": "OWNS"},
      {"from": "Q1 Roadmap", "to": "Product Launch", "type": "CONTAINS"}
    ]
  },
  "domains": ["STRATEGY", "EXECUTION"]
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Graph identifier |
| `description` | string | No | Human-readable description |
| `seed_data` | object | Yes | Entities and relationships to build from |
| `domains` | string[] | No | Restrict to specific domain layers |

**Response (201):**

```json
{
  "ok": true,
  "data": {
    "graph_id": "kg_1m8k3f7a2b9c",
    "name": "my-project-graph",
    "node_count": 3,
    "edge_count": 2,
    "domains": ["STRATEGY", "EXECUTION"],
    "status": "built",
    "created_at": "2026-02-25T12:00:00.000Z"
  }
}
```

---

### POST /api/v2/memory/design

Design a memory architecture for an AI agent, including layer structure, retention policies, and retrieval strategies.

**Price:** $25.00 per request

**Request:**

```json
{
  "agent_type": "trading_bot",
  "requirements": {
    "persistence": "long_term",
    "retrieval": "semantic_search",
    "max_nodes": 10000,
    "domains": ["EXECUTION", "TRACK"]
  },
  "constraints": {
    "latency_ms": 100,
    "storage_gb": 5
  }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `agent_type` | string | Yes | Type of agent this architecture serves |
| `requirements` | object | Yes | Architecture requirements |
| `constraints` | object | No | Performance and resource constraints |

**Response (200):**

```json
{
  "ok": true,
  "data": {
    "design_id": "md_1m8k3f7a2b9c",
    "architecture": {
      "layers": [
        {
          "name": "working_memory",
          "type": "in_memory",
          "ttl_seconds": 3600,
          "description": "Short-term context for active trades"
        },
        {
          "name": "episodic_memory",
          "type": "neo4j",
          "retention": "90_days",
          "description": "Trade history and decision records"
        },
        {
          "name": "semantic_memory",
          "type": "vector_index",
          "embedding_model": "text-embedding-3-small",
          "description": "Pattern library and market knowledge"
        }
      ],
      "retrieval_strategy": "hybrid_bm25_vector",
      "estimated_latency_ms": 45,
      "estimated_storage_gb": 2.1
    },
    "created_at": "2026-02-25T12:00:00.000Z"
  }
}
```

---

### GET /api/token/price

Live $ODAI token price from DexScreener. No authentication required. Cached for 60 seconds.

**Response (200):**

```json
{
  "ok": true,
  "data": {
    "price": "0.0000142",
    "mcap": 1500000,
    "volume24h": 250000,
    "change24h": 5.2,
    "change1h": -0.8,
    "buys24h": 340,
    "sells24h": 180
  }
}
```

Response headers:
- `X-Cache: HIT` or `MISS`
- `Cache-Control: public, max-age=60`

---

### GET /api/context

Live project context snapshot. No authentication required. Updated every 5 minutes by the context-updater daemon.

**Response (200):**

```json
{
  "ok": true,
  "project": "ODEI",
  "updated_at": "2026-02-25T12:00:00.000Z",
  "price": { ... },
  "daemons": { ... },
  "exchanges": { ... }
}
```

---

### GET /health

Service health check. No authentication required.

**Response (200):**

```json
{
  "ok": true,
  "service": "odei-api",
  "version": "1.0.0",
  "mode": "public-web",
  "startedAt": "2026-02-25T08:00:00.000Z",
  "uptime": 14400,
  "uptimeSeconds": 14400,
  "timestamp": "2026-02-25T12:00:00.000Z"
}
```

---

### GET /api/v2/health

Extended health check with upstream graph status. No authentication required.

**Response (200):**

```json
{
  "ok": true,
  "data": {
    "service": "odei-api-v2",
    "status": "healthy",
    "version": "2.0.0",
    "uptime_seconds": 14400,
    "started_at": "2026-02-25T08:00:00.000Z",
    "timestamp": "2026-02-25T12:00:00.000Z",
    "graph": {
      "node_count": 8147,
      "edge_count": 12340,
      "domain_counts": {
        "FOUNDATION": 42,
        "VISION": 18,
        "STRATEGY": 23,
        "TACTICS": 31,
        "EXECUTION": 15,
        "TRACK": 13
      },
      "last_updated": "2026-02-25T10:00:00.000Z"
    },
    "upstream": {
      "worldmodel_reachable": true,
      "worldmodel_latency_ms": 45
    }
  }
}
```

---

### GET /api/v2/schema

World model schema: node types, relationship types, and domain definitions. No authentication required.

**Response (200):**

```json
{
  "ok": true,
  "data": {
    "schema_version": "1.3.0",
    "generated_at": "2026-02-22T13:31:43.493Z",
    "node_type_count": 59,
    "relationship_type_count": 92,
    "domains": [
      { "name": "FOUNDATION", "description": "Identity, values, principles, guardrails, partnerships", "node_count": 42 },
      { "name": "VISION", "description": "Goals, objectives, key results", "node_count": 18 },
      { "name": "STRATEGY", "description": "Strategies, milestones, risks", "node_count": 23 },
      { "name": "TACTICS", "description": "Projects, initiatives", "node_count": 31 },
      { "name": "EXECUTION", "description": "Tasks, decisions, actions, work sessions", "node_count": 15 },
      { "name": "TRACK", "description": "Metrics, signals, audit log, evidence", "node_count": 13 }
    ],
    "node_types": [
      { "name": "Guardrail", "layer": "policy", "required_properties": ["response", "summary", "title", "trigger"] },
      { "name": "Goal", "layer": "vision", "required_properties": ["summary", "title"] },
      ...
    ],
    "relationship_types": [
      { "name": "ENFORCES", "from_layer": "policy", "to_layer": "foundation", "directional": true, "cardinality": "N:M" },
      { "name": "PURSUES_GOAL", "from_layer": "foundation", "to_layer": "vision", "directional": true, "cardinality": "1:N" },
      ...
    ]
  }
}
```

---

### POST /api/intake

Register for ODEI onboarding. No authentication required. Rate-limited.

**Request:**

```json
{
  "lane": "agent",
  "name": "My Trading Bot",
  "contact": "dev@example.com"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `lane` | string | Yes | `agent`, `human`, or `enterprise` |
| `name` | string | No | Display name |
| `contact` | string | No | Contact email or handle |

**Response (201):**

```json
{
  "ok": true,
  "intake_id": "intake_1m8k3f7a2b9c",
  "lane": "agent",
  "status": "registered",
  "timestamp": "2026-02-25T12:00:00.000Z"
}
```

---

### Solana Actions (Blinks)

Endpoints follow the [Solana Actions](https://solana.com/docs/advanced/actions) standard.

#### GET /blink/guardrail

Returns the Action metadata (icon, title, description, input fields).

#### POST /blink/guardrail

Execute a guardrail check via Solana Actions. Returns a transaction for the user to sign.

#### GET /blink/worldmodel

Returns the Action metadata for world model queries.

#### POST /blink/worldmodel

Execute a world model query via Solana Actions.

---

## CORS

All endpoints include CORS headers:

```
Access-Control-Allow-Origin: *
Access-Control-Allow-Methods: GET, POST, OPTIONS
Access-Control-Allow-Headers: Content-Type, Authorization, X-Api-Key
```

Preflight `OPTIONS` requests return `204` with `Access-Control-Max-Age: 86400`.

---

## Webhooks

### ClawGig Webhook

**Endpoint:** `POST /api/clawgig-webhook`

Receives gig assignments from the ClawGig marketplace. Requires the `X-Webhook-Secret` header matching `CLAWGIG_WEBHOOK_SECRET`.

**Request:**

```json
{
  "type": "guardrail_check",
  "payload": { "action": "...", "context": "..." },
  "gig_id": "f7857b34-..."
}
```

**Response (200):**

```json
{
  "ok": true,
  "service": "guardrail_check",
  "verdict": "APPROVED",
  "reason": "Checked against ODEI constitutional framework"
}
```

---

## SDK / Client Libraries

Official examples are available in the [odei-ai/examples](https://github.com/odei-ai/examples) repository:

**Python:**

```python
import requests

resp = requests.post(
    "https://api.odei.ai/api/v2/guardrail/check",
    headers={"X-Api-Key": "YOUR_KEY"},
    json={"action": "Deploy new contract", "context": "Testnet deployment"}
)
print(resp.json())
```

**TypeScript:**

```typescript
const res = await fetch("https://api.odei.ai/api/v2/guardrail/check", {
  method: "POST",
  headers: {
    "Content-Type": "application/json",
    "X-Api-Key": "YOUR_KEY",
  },
  body: JSON.stringify({
    action: "Deploy new contract",
    context: "Testnet deployment",
  }),
});
const data = await res.json();
console.log(data);
```

**curl:**

```bash
curl -X POST https://api.odei.ai/api/v2/guardrail/check \
  -H "Content-Type: application/json" \
  -H "X-Api-Key: YOUR_KEY" \
  -d '{"action":"Deploy new contract","context":"Testnet deployment"}'
```
