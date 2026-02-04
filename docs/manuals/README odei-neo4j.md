# odei-neo4j MCP Server

MCP service that exposes curated graph reads and promotion tooling for the ODEI constitutional Neo4j database. Raw conversation logs never touch Neo4j; only validated canonical entities are written through these MCP tools.

## Quick start

```bash
cd servers/odei-neo4j
cp .env.example .env           # fill with your credentials
npm install
npm run build
npm run setup                  # applies constraints from src/cypher/constraints.cypher
npm start                      # serves JSON-RPC over stdio
```

During development, `npm run dev` runs the TypeScript entry with `tsx`.

## Environment

| Variable         | Required | Description                                                   |
| ---------------- | -------- | ------------------------------------------------------------- |
| `NEO4J_URI`      | Yes      | Bolt URI (`bolt://host:7687`).                                |
| `NEO4J_USERNAME` | Yes      | Database user with rights to read/write canonical graph data. |
| `NEO4J_PASSWORD` | Yes      | Secret credential (never commit).                             |
| `NEO4J_DATABASE` | Yes      | Database name (defaults to `neo4j`).                          |

Load these via `.env` or your process manager; credentials must **not** enter source control.

## JSON-RPC usage

```bash
printf '%s\n' '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{}}' \
  | node dist/index.js

printf '%s\n' '{"jsonrpc":"2.0","id":2,"method":"call_tool","params":{"name":"odei.decisions.list.v1","args":{}}}' \
  | node dist/index.js
```

Validation failures return an error object with `code:422` (`MCP_VALIDATION_FAILED`). Connection or driver errors use `code:-32000`.

## Tool surface (v1)

### Read tools

- `odei.discuss.topics.v1`
- `odei.discuss.threadById.v1`
- `odei.decisions.list.v1`
- `odei.decisions.roiById.v1`
- `odei.execute.tasksToday.v1`
- `odei.mind.patterns.v1`
- `odei.mind.metrics.v1`

Each returns curated graph data following the schema in [`schema.md`](./schema.md). Empty arrays are expected on a clean database.

### Promotion tools

- `odei.promote.validate.v1` — static validation of candidate entities/links (no writes).
- `odei.promote.create.v1` — MERGE canonical nodes/edges with provenance enforcement and idempotent behaviour.

Promotion requires inputs that already passed validation; only canonical facts enter the graph.

## Constraints

`npm run setup` executes Cypher statements in `src/cypher/constraints.cypher` to ensure unique IDs and required indexes. Run once per database or whenever new constraints are added.

## Claude Code configuration

```json
{
  "mcpServers": {
    "odei-neo4j": {
      "command": "node",
      "args": ["./servers/odei-neo4j/dist/index.js"],
      "cwd": ".",
      "env": {
        "NEO4J_URI": "bolt://localhost:7687",
        "NEO4J_USERNAME": "neo4j",
        "NEO4J_PASSWORD": "REDACTED",
        "NEO4J_DATABASE": "neo4j"
      }
    }
  }
}
```

## Smoke checklist

1. `npm run build` — compiles sources.
2. `npm run setup` — applies constraints successfully.
3. `printf '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{}}' | node dist/index.js` → `{ "ok": true }`.
4. `odei.decisions.list.v1` on empty DB returns `{ "items": [] }`.
5. `odei.promote.validate.v1` rejects incomplete payloads (`code:422`).
6. `odei.promote.create.v1` MERGE-ing a Goal + Project + HAS_PROJECT is idempotent across repeated calls.

Refer to [`schema.md`](./schema.md) for the seven-layer constitutional model and invariants enforced by the validation layer.

## Hybrid search payload controls

`odei.neo4j.hybrid.search.v1` and `odei.neo4j.hybrid.plusplus.search.v1` now return lean payloads by default to keep responses well below MCP token budgets. Both tools accept an optional `excludeFields` array (`["properties","metadata","provenance","tags"]`). Omitted or `undefined` uses the safe default of `["properties","metadata","provenance"]`. Pass `[]` explicitly if you truly need every field (embeddings are always stripped for safety). Example:

```json
{
  "name": "odei.neo4j.hybrid.search.v1",
  "arguments": {
    "query": "10Y vision runway",
    "topK": 5,
    "excludeFields": ["metadata"] // keep properties + provenance, drop metadata
  }
}
```

Context nodes are always lightweight (no metadata/properties) so downstream UI panels stay responsive.

### Query semantics & guardrails

Both `odei.neo4j.hybrid.search.v1` and `odei.neo4j.hybrid.plusplus.search.v1` share the following defaults:

- Multi-word queries are tokenized automatically and expanded into Lucene-compatible strings, so `"family commitment"` matches both the phrase and its individual words even if the full-text index lacks the exact literal.
- `minScore` now defaults to **0.30**, which captures ~95% of the curated corpus without flooding the UI. Pass a higher value explicitly if you need stricter recall.
- `topK` is clamped to **100** to prevent unbounded memory growth; the server still fetches a small superset (2×) internally so Reciprocal Rank Fusion has enough candidates.
- Both semantic and structural branches share the same token pipeline, which keeps CONTAINS fallbacks consistent with full-text behavior.

## Write tool parameter reference

All write tools enforce provenance and Guardian rules. Every request must include a `provenance` block with `{module, actor, source, confidence}`; any payload touching the Foundation layer requires `actor: "human"` or `"joint"`.

### Node create (`odei.neo4j.value.create.v1`, etc.)

```json
{
  "name": "odei.neo4j.value.create.v1",
  "arguments": {
    "data": {
      "title": "Freedom over security",
      "summary": "Default stance",
      "description": "Detailed context",
      "priority": "core"
    },
    "provenance": {
      "module": "discuss",
      "actor": "human",
      "source": "tool_test",
      "confidence": 1
    },
    "activateImmediately": false,
    "idempotencyKey": "optional-uuid"
  }
}
```

Nodes are created as `draft` and promoted to `active` automatically once embeddings are generated (synchronous if `activateImmediately` is true, otherwise background).

### Node update (`odei.neo4j.goal.update.v1`, etc.)

```json
{
  "name": "odei.neo4j.goal.update.v1",
  "arguments": {
    "id": "goal_q1_launch",
    "data": {
      "description": "New details go here",
      "summary": "One-line summary"
    },
    "provenance": {
      "module": "decisions",
      "actor": "agent",
      "source": "retro_adjustment",
      "confidence": 0.9
    }
  }
}
```

At least one field inside `data` is required. Guardian enforces layer/type consistency automatically.

### Relationship create (`odei.neo4j.relationship.create.v1`)

```json
{
  "name": "odei.neo4j.relationship.create.v1",
  "arguments": {
    "type": "SERVES",
    "fromId": "goal_q1_launch",
    "toId": "vision_10y_transform",
    "properties": {
      "weight": 0.85,
      "notes": "OKR linkage"
    },
    "provenance": {
      "module": "execute",
      "actor": "joint",
      "source": "alignment_review",
      "confidence": 0.95
    }
  }
}
```

Relationships inherit layer/type guards automatically. Use `weight` for `SUPPORTED_BY`, `ALIGNS_WITH`, `ENFORCES`, and `sim` for `SIMILAR_TO` if required.
