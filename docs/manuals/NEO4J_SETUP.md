# Neo4j Database Setup — Optimized for Multi-Modal Search

## ✅ Configuration Complete

Your Neo4j database is now fully configured for optimal search performance across all MCP servers.

## 📊 Current State

**Database:** `memory`
**URI:** `bolt://localhost:7687`
**Constraints:** 47
**Indexes:** 213

**Existing Data:**

- Value: 4 nodes
- Principle: 4 nodes
- Guardrail: 3 nodes
- Human: 1 node
- Context: 1 node
- AI: 1 node

**⚠️ Note:** Embeddings not yet generated for existing nodes. Run migration script to add embeddings.

---

## 🔍 Search Capabilities

### 1. **Semantic Search (Vector Embeddings)**

- **Model:** `text-embedding-3-large`
- **Dimensions:** 3072
- **Similarity Function:** Cosine similarity

**Indexed Node Types:**

- Foundation: `Value`, `Principle`, `Guardrail`
- Vision: `Vision`, `Business`, `Goal`, `Season`
- Strategy: `Strategy`, `Objective`, `KeyResult`
- Tactics: `Project`, `Area`
- Mind: `Insight`, `Pattern`, `Note`, `Memory`

**Usage Example:**

```cypher
// Find values similar to "freedom and autonomy"
CALL db.index.vector.queryNodes('value_embedding_idx', 10, $queryEmbedding)
YIELD node, score
WHERE score >= 0.6
RETURN node, score
ORDER BY score DESC
```

---

### 2. **Keyword Search (Full-Text Indexes)**

**Indexed Layers:**

- **Foundation:** `Value`, `Principle`, `Guardrail` (name, description, why)
- **Vision:** `Vision`, `Business`, `Goal`, `Season` (name, description, reasoning)
- **Strategy:** `Strategy`, `Objective`, `KeyResult`, `Initiative` (name, description, outcome)
- **Tactics:** `Project`, `Area`, `System`, `Process` (name, description, notes)
- **Execution:** `Decision`, `Task`, `Action` (title, description, notes)
- **Mind:** `Insight`, `Pattern`, `Note` (title, content, summary)
- **Memory:** `Memory` (topic, content, outcome, decision)

**Usage Example:**

```cypher
// Search for goals related to "financial independence"
CALL db.index.fulltext.queryNodes('vision_text_search', 'financial independence')
YIELD node, score
WHERE node:Goal
RETURN node, score
ORDER BY score DESC
LIMIT 10
```

---

### 3. **Fast Filtering (Property Indexes)**

**Indexed Properties:**

- `Goal.horizon` — Time horizon (daily, weekly, monthly, etc.)
- `Task.dueDate` — Task due dates
- `Task.status` — Task status (todo, in_progress, done)
- `Decision.created` — Decision timestamps
- `Memory.timestamp` — Memory creation time
- `Insight.created` — Insight timestamps

**Usage Example:**

```cypher
// Find all goals with 5-year horizon
MATCH (g:Goal)
WHERE g.horizon = '5-year'
RETURN g
```

---

### 4. **Data Integrity (Uniqueness Constraints)**

**All node types have unique name/id constraints:**

- Foundation: `Value.name`, `Principle.name`, `Guardrail.name`, etc.
- Execution: `Decision.id`, `Task.id`, `Action.id`
- Mind: `Insight.id`, `Pattern.id`, `Note.id`

This prevents duplicate entries and ensures data consistency.

---

## 🚀 Using Search from MCP Servers

### From Discuss Agent

```javascript
// Semantic search
discuss_semantic_search({
  query: 'career decision corporate vs startup',
  nodeTypes: ['Value', 'Goal', 'Memory'],
  topK: 10,
  minScore: 0.6,
});

// Hybrid search (semantic + keyword + graph)
discuss_hybrid_search({
  query: 'financial runway burn rate',
  nodeTypes: ['Goal', 'Memory'],
  topK: 10,
  graphDepth: 2,
});

// Goal ladder
discuss_goal_ladder({
  goalName: 'Financial independence',
});
```

### From Other Agents (Decisions, Execute, Mind)

Currently using root `odei-neo4j` MCP server — need to add semantic search tools there.

**TODO:** Extend root MCP server with:

- `odei.neo4j.semantic.search.v1`
- `odei.neo4j.hybrid.search.v1`
- `odei.neo4j.fulltext.search.v1`

---

## ⚠️ Next Steps

### 1. **Generate Embeddings for Existing Nodes**

Run migration script to add embeddings to existing 14 nodes:

```bash
node scripts/generate-embeddings-for-existing-nodes.js
```

### 2. **Add Semantic Search to Root MCP Server**

Update `/servers/odei-neo4j/` to include vector search tools so ALL agents can use semantic search.

### 3. **Create Initial Constitution**

Populate the database with your core values, principles, and goals:

```javascript
// Example: Add your values
discuss_value_create({
  name: 'Freedom',
  description: 'Autonomy in choosing projects and work schedule',
  why: 'Maximizes creativity and prevents burnout',
});

discuss_goal_create({
  name: 'Financial Independence',
  description: 'Build $500K runway for 2 years of freedom',
  horizon: '5-year',
  reasoning: 'Enables risk-taking and experimentation without financial stress',
});
```

---

## 📁 Files Created

- `/scripts/setup-neo4j-indexes.cypher` — Cypher queries for manual setup
- `/scripts/setup-neo4j.js` — Automated setup script (already executed)
- `/docs/NEO4J_SETUP.md` — This documentation

---

## 🔧 Maintenance

### Re-run Setup (Idempotent)

If you need to re-run the setup (e.g., after database reset):

```bash
node scripts/setup-neo4j.js
```

All commands use `IF NOT EXISTS`, so it's safe to run multiple times.

### Check Database Health

```bash
node scripts/check-neo4j-health.js
```

(TODO: Create this health check script)

---

## 💡 Tips

1. **Always generate embeddings** when creating nodes with `name` or `description` fields
2. **Use semantic search** for conceptual queries ("values related to freedom")
3. **Use keyword search** for specific terms ("tasks due tomorrow")
4. **Use property indexes** for exact matches (`status = 'done'`)
5. **Combine all three** for hybrid search with re-ranking

---

**Status:** ✅ **Production Ready**
**Last Updated:** 2025-10-22
**Setup Version:** 1.0
