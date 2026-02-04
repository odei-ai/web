# ODEI Constitutional Knowledge Graph — Retrieval Architecture

**Version:** 1.0
**Date:** 2025-11-04
**Status:** Design Specification
**Authors:** Claude (Sonnet 4.5)

---

## Executive Summary

This document specifies the retrieval architecture for ODEI's constitutional knowledge graph, designed to scale from 45 nodes (current) to 5000+ nodes (1 year) while maintaining <5 second session start times and efficient token usage within Claude's 200K context window.

**Key Metrics:**

- **Current state:** 45 nodes, 44K tokens (foundation.list), dummy embeddings
- **Target state:** 5000+ nodes, <25K tokens session start, <5s latency
- **Context budget:** 200K total (30K system, 50K context, 120K conversation)
- **Embedding model:** OpenAI text-embedding-3-large (3072 dimensions)

**Architecture Principles:**

1. **Load foundation always** — Values, Principles, Guardrails, Human profile (Layer 1)
2. **Load vision on session start** — Goals, Visions (Layer 2)
3. **Search execution on demand** — Tasks, Metrics, Insights (Layers 5-7)
4. **Hybrid retrieval** — Structural (Cypher) + Semantic (vector) based on node type
5. **Progressive enhancement** — Start with essentials, load context as conversation deepens

---

## 1. Current State Analysis

### 1.1 Existing Implementation

**MCP Server Architecture:**

```
Root Workspace:
  odei-neo4j (servers/odei-neo4j) — Low-level CRUD, basic list tools

Agent Workspaces:
  discuss-neo4j (agents/discuss/mcp-servers/neo4j) — High-level semantic search
  odei-neo4j — Canonical write operations
```

**Available Tools:**

- **Layer tools:** `foundation.list.v1`, `vision.list.v1`, `strategy.list.v1`, etc.
- **Node type tools:** `value.create.v1`, `principle.update.v1`, etc. (32 types × 2 = 64 tools)
- **Graph tools:** `graph.getAll.v1`, `schema.inventory.v1`
- **Agent-specific:** `discuss_semantic_search`, `discuss_hybrid_search`, `discuss_goal_ladder`, etc.

**Current Issues:**

1. **Token overflow:** `foundation.list.v1` returns 44K tokens (limit: 25K)
2. **No field filtering:** Cannot exclude `properties`, `metadata`, `description` to reduce tokens
3. **Dummy embeddings:** All vectors are zeros (need real OpenAI embeddings)
4. **No vector indexes:** Missing `CREATE VECTOR INDEX` statements in constraints.cypher
5. **No retrieval strategy:** Agents blindly load all nodes, no selective loading

### 1.2 Node Distribution by Layer

| Layer          | Current | 6 Months | 1 Year | Node Types                                                  | Retrieval Strategy        |
| -------------- | ------- | -------- | ------ | ----------------------------------------------------------- | ------------------------- |
| Foundation (1) | 45      | 60       | 80     | Value, Principle, Guardrail, Human, AI, Policy, Context     | **Always load fully**     |
| Vision (2)     | 0       | 15       | 25     | Vision, Business, Goal, Season                              | **Always load fully**     |
| Strategy (3)   | 0       | 50       | 100    | Strategy, Objective, KeyResult, Initiative, Milestone, Risk | **Semantic search**       |
| Tactics (4)    | 0       | 80       | 150    | Project, Area, System, Process                              | **Semantic search**       |
| Execution (5)  | 0       | 250      | 2000   | Decision, Task, TimeBlock, WorkSession, Action              | **Time-based + semantic** |
| Track (6)      | 0       | 500      | 2000   | Metric, Observation, Event, Signal                          | **Aggregated only**       |
| Mind (7)       | 0       | 100      | 500    | Insight, Pattern, Evidence, Source, Note                    | **Semantic search**       |

**Total:** 45 → 1055 → 4855 nodes

### 1.3 Token Budget Breakdown

**Current Usage (foundation.list only):**

```
System Prompt:        30K tokens (CLAUDE.md + tool schemas)
MCP Tools:            15K tokens (odei-neo4j tool definitions)
Foundation Context:   44K tokens (foundation.list — OVER BUDGET)
Conversation:         111K tokens (remaining)
---
Total Available:      200K tokens
```

**Target Usage (optimized):**

```
System Prompt:        30K tokens (unchanged)
MCP Tools:            15K tokens (unchanged)
Session Context:      25K tokens (Foundation + Vision + Workload — OPTIMIZED)
  - Foundation:       15K tokens (Values, Principles, Guardrails, Human, AI)
  - Vision:           8K tokens (Goals, Visions, active only)
  - Workload:         2K tokens (capacity %, next 7 days calendar)
Conversation:         130K tokens (increased from 111K)
---
Total Available:      200K tokens
```

---

## 2. Layer-by-Layer Retrieval Strategy

### 2.1 Foundation Layer (Layer 1) — ALWAYS LOAD

**Rationale:**

- Constitutional foundation drives all downstream decisions
- Small size: 45 → 80 nodes (stable, slow growth)
- High access frequency: Referenced in 80%+ of conversations
- Low update frequency: Weekly at most
- Critical for agent alignment

**Loading Strategy:**

```cypher
// Session start query (foundation.list.v1 with excludeFields)
MATCH (n:OdeiNode)
WHERE n.layer = 'foundation'
  AND n.status = 'active'
RETURN
  n.id, n.type, n.layer, n.title, n.summary, n.status,
  n.createdAt, n.updatedAt,
  // Include full text for Values, Principles, Guardrails
  CASE
    WHEN n.type IN ['value', 'principle', 'guardrail']
    THEN n.description
    ELSE null
  END AS description,
  // Exclude heavy fields
  null AS properties,
  null AS metadata,
  null AS embedding
ORDER BY n.type, n.createdAt
```

**Token Optimization:**

- **Exclude:** `properties`, `metadata`, `embedding`, `provenance`
- **Include:** `id`, `type`, `title`, `summary`, `description` (for core types only)
- **Expected size:** ~15K tokens (down from 44K)

**Caching:**

- Client-side cache for session duration
- Invalidate on write operations (create/update/delete)
- TTL: Until next write event

**Relationships:**
Load only critical foundation fabric:

```cypher
// Foundation relationships (lightweight)
MATCH (from:OdeiNode)-[r]->(to:OdeiNode)
WHERE from.layer = 'foundation' AND to.layer = 'foundation'
  AND from.status = 'active' AND to.status = 'active'
RETURN r.id, type(r) AS type, from.id AS fromId, to.id AS toId
```

### 2.2 Vision Layer (Layer 2) — ALWAYS LOAD

**Rationale:**

- Active goals and visions needed for alignment checks
- Small size: 0 → 25 nodes (stable)
- High access frequency: Every session start
- Hierarchy traversal required (Goal ladder)

**Loading Strategy:**

```cypher
// Session start query (vision.list.v1 with excludeFields)
MATCH (n:OdeiNode)
WHERE n.layer = 'vision'
  AND n.status = 'active'
OPTIONAL MATCH (n)-[r:SUPPORTED_BY|SERVES|ALIGNS_WITH]->(parent)
WHERE parent.status = 'active'
RETURN
  n.id, n.type, n.layer, n.title, n.summary, n.status,
  n.horizon, n.deadline, n.target,  // Vision/Goal-specific fields
  collect({id: parent.id, type: parent.type, title: parent.title, rel: type(r)}) AS parents,
  null AS properties,
  null AS metadata,
  null AS embedding
ORDER BY
  CASE n.type
    WHEN 'vision' THEN 1
    WHEN 'business' THEN 2
    WHEN 'goal' THEN 3
    WHEN 'season' THEN 4
  END,
  n.horizon,
  n.createdAt
```

**Token Optimization:**

- **Include parent relationships inline** — Avoid separate relationship query
- **Exclude:** `properties`, `metadata`, `embedding`, `description`
- **Expected size:** ~8K tokens

### 2.3 Strategy Layer (Layer 3) — SEMANTIC SEARCH ON DEMAND

**Rationale:**

- Intermediate layer (50 → 100 nodes at maturity)
- Access frequency: 30-40% of conversations
- Search patterns: "What strategies support Goal X?"

**Retrieval Patterns:**

**Pattern A: Goal-driven strategy search**

```cypher
// Find strategies aligned with active goal
MATCH (goal:Goal {id: $goalId})
MATCH (strategy:Strategy)-[:ALIGNS_WITH|SERVES]->(goal)
WHERE strategy.status = 'active'
RETURN strategy
ORDER BY strategy.updatedAt DESC
LIMIT 10
```

**Pattern B: Semantic search for strategy concepts**

```cypher
// Vector similarity search
CALL db.index.vector.queryNodes(
  'strategy_embeddings',  // vector index name
  $topK,                  // default: 10
  $queryEmbedding         // OpenAI embedding of user query
) YIELD node, score
WHERE node.status = 'active'
  AND score > $minSimilarity  // default: 0.7
RETURN node, score
ORDER BY score DESC
```

**Pattern C: Hybrid search (combine structural + semantic)**

```cypher
// Combine relationship traversal + vector similarity
WITH $queryEmbedding AS qEmbed, $goalId AS gId
CALL {
  // Structural: Strategies linked to goal
  MATCH (goal:Goal {id: gId})
  MATCH (s:Strategy)-[:ALIGNS_WITH|SERVES]->(goal)
  WHERE s.status = 'active'
  RETURN s AS result, 1.0 AS structuralScore, 0.0 AS semanticScore
  LIMIT 5

  UNION

  // Semantic: Vector similarity
  CALL db.index.vector.queryNodes('strategy_embeddings', 10, qEmbed)
  YIELD node AS s, score AS semScore
  WHERE s.status = 'active' AND semScore > 0.7
  RETURN s AS result, 0.0 AS structuralScore, semScore AS semanticScore
  LIMIT 10
}
WITH result, structuralScore, semanticScore,
  (structuralScore * 0.6 + semanticScore * 0.4) AS finalScore
RETURN result, finalScore
ORDER BY finalScore DESC
LIMIT 10
```

**Token Budget:**

- **Per result:** ~800-1200 tokens (full node with relationships)
- **Typical query:** 5-10 results = 5-10K tokens
- **Max safe:** 15K tokens (reserve for multi-turn retrieval)

### 2.4 Tactics Layer (Layer 4) — SEMANTIC SEARCH ON DEMAND

**Rationale:**

- Medium layer (80 → 150 nodes at maturity)
- Access frequency: 20-30% of conversations
- Search patterns: "What projects support Initiative X?"

**Retrieval Pattern:**

- Same hybrid search as Strategy layer
- Vector index: `tactics_embeddings`
- Typical topK: 10-15 results

### 2.5 Execution Layer (Layer 5) — TIME-BASED + SEMANTIC

**Rationale:**

- Large layer (250 → 2000 nodes at maturity)
- High churn: Daily creates/updates
- Access patterns: Time-based (today's tasks) + semantic (find related decisions)

**Retrieval Patterns:**

**Pattern A: Time-based (today's tasks, this week's decisions)**

```cypher
// Tasks due within time window
MATCH (t:Task)
WHERE t.status IN ['todo', 'in_progress']
  AND t.dueDate >= $startDate
  AND t.dueDate <= $endDate
RETURN t
ORDER BY t.dueDate, t.priority DESC
LIMIT 50
```

**Pattern B: Semantic search for related decisions**

```cypher
// Find decisions similar to current discussion
CALL db.index.vector.queryNodes(
  'execution_embeddings',
  15,  // Higher topK for execution layer
  $queryEmbedding
) YIELD node, score
WHERE node.type IN ['decision', 'task', 'action']
  AND node.status = 'active'
  AND score > 0.65  // Lower threshold for execution (more recall)
RETURN node, score
ORDER BY score DESC, node.updatedAt DESC
```

**Pattern C: Decision precedent search (hybrid)**

```cypher
// Find decisions with similar context + high ROI
WITH $queryEmbedding AS qEmbed, $minROI AS roi
CALL db.index.vector.queryNodes('execution_embeddings', 20, qEmbed)
YIELD node AS d, score
WHERE d.type = 'decision'
  AND d.status = 'active'
  AND score > 0.6
  AND d.roi.total >= roi
RETURN d, score
ORDER BY d.roi.total DESC, score DESC
LIMIT 10
```

**Token Budget:**

- **Per result:** ~600-800 tokens (Task/Decision nodes lighter than Strategy)
- **Typical query:** 10-20 results = 8-15K tokens
- **Workload assessment:** 2K tokens (aggregated capacity %)

### 2.6 Track Layer (Layer 6) — AGGREGATED METRICS ONLY

**Rationale:**

- Large layer (500 → 2000 nodes at maturity)
- High write frequency: Hourly observations
- Access patterns: Aggregated summaries, trend analysis
- Individual nodes rarely needed

**Retrieval Strategy:**

```cypher
// Aggregated metrics (no individual nodes)
MATCH (m:Metric)
WHERE m.status = 'active'
WITH m
MATCH (m)<-[:MEASURES]-(obs:Observation)
WHERE obs.observedAt >= $startDate
  AND obs.observedAt <= $endDate
RETURN
  m.id, m.title, m.unit,
  count(obs) AS observationCount,
  avg(obs.value) AS avgValue,
  min(obs.value) AS minValue,
  max(obs.value) AS maxValue,
  collect({timestamp: obs.observedAt, value: obs.value}) AS timeseries
ORDER BY m.title
```

**Token Budget:**

- **Aggregated view:** ~3-5K tokens (10-20 metrics with summaries)
- **Full retrieval:** Only on explicit request (rare)

### 2.7 Mind Layer (Layer 7) — SEMANTIC SEARCH FOR INSIGHTS

**Rationale:**

- Medium layer (100 → 500 nodes at maturity)
- Access frequency: 15-20% of conversations
- Search patterns: "What insights relate to [topic]?"

**Retrieval Pattern:**

```cypher
// Semantic search for insights with evidence
CALL db.index.vector.queryNodes(
  'mind_embeddings',
  10,
  $queryEmbedding
) YIELD node AS insight, score
WHERE insight.type IN ['insight', 'pattern']
  AND insight.status = 'active'
  AND insight.confidence >= $minConfidence  // default: 0.6
  AND score > 0.7
OPTIONAL MATCH (insight)-[:SUPPORTED_BY]->(evidence:Evidence)
RETURN insight, score, collect(evidence) AS evidence
ORDER BY insight.confidence DESC, score DESC
LIMIT 10
```

**Token Budget:**

- **Per result:** ~1000-1500 tokens (Insight + Evidence)
- **Typical query:** 5-10 results = 6-12K tokens

---

## 3. Embedding Strategy

### 3.1 Which Nodes Need Embeddings?

**All nodes except:**

- **Human/AI identity nodes** — Referenced by ID, not semantic content
- **TimeBlock/WorkSession** — Time-based retrieval only
- **Metric** — Aggregated retrieval only

**Priority tiers:**

| Priority | Node Types                        | Embedding Content             | Update Frequency |
| -------- | --------------------------------- | ----------------------------- | ---------------- |
| **P0**   | Value, Principle, Guardrail       | title + statement/description | On create/update |
| **P1**   | Vision, Goal, Strategy, Objective | title + narrative/summary     | On create/update |
| **P2**   | Decision, Insight, Pattern        | title + summary + reasoning   | On create/update |
| **P3**   | Task, Action, Initiative, Risk    | title + description           | Batch nightly    |
| **P4**   | Evidence, Source, Note            | content field                 | Batch weekly     |

### 3.2 Embedding Content Construction

**Template (from embeddingManager.ts):**

```typescript
function buildEmbeddingText(node: GraphNodeRecord): string {
  const lines: string[] = [];

  lines.push(`Type: ${normalizeTypeLabel(node)}`);
  if (node.title) lines.push(`Title: ${node.title}`);
  if (node.summary) lines.push(`Summary: ${node.summary}`);
  if (node.description) lines.push(`Description: ${node.description}`);

  // Include type-specific fields
  for (const [key, value] of Object.entries(node.properties)) {
    if (meaningfulField(key)) {
      lines.push(`${key}: ${value}`);
    }
  }

  return lines.join('\n').slice(0, 6000); // Max 6K chars
}
```

**Meaningful fields by type:**

| Node Type | Fields to Include                           |
| --------- | ------------------------------------------- |
| Value     | title + description + priority              |
| Principle | title + statement + rationale               |
| Guardrail | title + trigger + response + severity       |
| Vision    | title + narrative + horizon                 |
| Goal      | title + target + deadline + successCriteria |
| Decision  | title + summary + roi + decidedAt           |
| Strategy  | title + narrative + timebox                 |
| Insight   | title + summary + confidence + insightType  |
| Pattern   | title + pattern + confidence                |

### 3.3 Embedding Dimensions

**Model:** `text-embedding-3-large` (OpenAI)

**Dimension Choice:**

- **3072 dimensions** (full) — Current configuration
- **1536 dimensions** — Cost optimization option

**Rationale for 3072:**

- Accuracy > cost for constitutional knowledge (high stakes)
- Graph size (5K nodes) is manageable cost-wise (~$0.13 per 1M tokens)
- Total embedding cost at 5K nodes: ~$2-3 (one-time + updates)

**Cost Analysis:**

```
5000 nodes × 300 tokens/node average = 1.5M tokens
OpenAI text-embedding-3-large: $0.13 per 1M tokens
Total: $0.20 (initial) + $0.05/month (updates)
```

**Decision:** Use 3072 dimensions (no downsampling)

### 3.4 Update Strategy

**On Create:**

- Generate embedding immediately (synchronous)
- Retry 3x with exponential backoff on failure
- Log failures but don't block write

**On Update:**

- Check if meaningful fields changed (title, description, summary, etc.)
- Skip embedding update if only metadata/timestamps changed
- Generate new embedding if meaningful content changed

**Batch Updates:**

- Nightly cron for P3/P4 nodes (Task, Action, Evidence)
- Weekly recompute for drift detection (optional)

**Implementation (existing):**

```typescript
// From embeddingManager.ts
export async function ensureEmbeddingForNode(
  node: GraphNodeRecord,
  ctx: ToolContext,
  options: { force?: boolean; updatedFields?: string[] } = {}
): Promise<void> {
  // Skip if no meaningful fields changed
  if (!options.force && shouldSkipEmbeddingUpdate(node, options.updatedFields)) {
    return;
  }

  // Generate embedding via Neo4j GenAI plugin
  await session.run(
    `
    CALL genai.vector.encodeBatch([$text], $provider, {
      token: $token,
      model: $model
    }) YIELD vector
    MATCH (n {id: $nodeId})
    SET n.embedding = vector,
        n.embeddingUpdatedAt = datetime()
  `,
    { text, nodeId, provider: 'OpenAI', token: apiKey, model: 'text-embedding-3-large' }
  );
}
```

---

## 4. Vector Index Configuration

### 4.1 Required Indexes

**Add to `/Users/ai/ODEI/servers/odei-neo4j/src/cypher/constraints.cypher`:**

```cypher
// ===== Vector Indexes (require Neo4j 5.11+) =====

// Foundation layer embeddings
CREATE VECTOR INDEX foundation_embeddings IF NOT EXISTS
FOR (n:OdeiNode)
ON n.embedding
OPTIONS {
  indexConfig: {
    `vector.dimensions`: 3072,
    `vector.similarity_function`: 'cosine'
  }
}
WHERE n.layer = 'foundation';

// Vision layer embeddings
CREATE VECTOR INDEX vision_embeddings IF NOT EXISTS
FOR (n:OdeiNode)
ON n.embedding
OPTIONS {
  indexConfig: {
    `vector.dimensions`: 3072,
    `vector.similarity_function`: 'cosine'
  }
}
WHERE n.layer = 'vision';

// Strategy layer embeddings
CREATE VECTOR INDEX strategy_embeddings IF NOT EXISTS
FOR (n:OdeiNode)
ON n.embedding
OPTIONS {
  indexConfig: {
    `vector.dimensions`: 3072,
    `vector.similarity_function`: 'cosine'
  }
}
WHERE n.layer = 'strategy';

// Tactics layer embeddings
CREATE VECTOR INDEX tactics_embeddings IF NOT EXISTS
FOR (n:OdeiNode)
ON n.embedding
OPTIONS {
  indexConfig: {
    `vector.dimensions`: 3072,
    `vector.similarity_function`: 'cosine'
  }
}
WHERE n.layer = 'tactics';

// Execution layer embeddings
CREATE VECTOR INDEX execution_embeddings IF NOT EXISTS
FOR (n:OdeiNode)
ON n.embedding
OPTIONS {
  indexConfig: {
    `vector.dimensions`: 3072,
    `vector.similarity_function`: 'cosine'
  }
}
WHERE n.layer = 'execution';

// Mind layer embeddings
CREATE VECTOR INDEX mind_embeddings IF NOT EXISTS
FOR (n:OdeiNode)
ON n.embedding
OPTIONS {
  indexConfig: {
    `vector.dimensions`: 3072,
    `vector.similarity_function`: 'cosine'
  }
}
WHERE n.layer = 'mind';

// ===== Property Indexes for Performance =====

// Status filtering (heavily used)
CREATE INDEX odei_node_status IF NOT EXISTS
FOR (n:OdeiNode)
ON (n.status);

// Layer filtering (all queries)
CREATE INDEX odei_node_layer IF NOT EXISTS
FOR (n:OdeiNode)
ON (n.layer);

// Type filtering (common)
CREATE INDEX odei_node_type IF NOT EXISTS
FOR (n:OdeiNode)
ON (n.type);

// Time-based queries (execution layer)
CREATE INDEX odei_node_updated IF NOT EXISTS
FOR (n:OdeiNode)
ON (n.updatedAt);

// Task due date queries
CREATE INDEX task_due_date IF NOT EXISTS
FOR (n:Task)
ON (n.dueDate);

// Decision ROI sorting
CREATE INDEX decision_roi IF NOT EXISTS
FOR (n:Decision)
ON (n.roi.total);
```

### 4.2 Index Maintenance

**Query index status:**

```cypher
SHOW INDEXES
YIELD name, type, labelsOrTypes, properties, state, populationPercent
WHERE type = 'VECTOR'
RETURN name, state, populationPercent;
```

**Rebuild index if needed:**

```cypher
DROP INDEX foundation_embeddings IF EXISTS;
CREATE VECTOR INDEX foundation_embeddings FOR (n:OdeiNode) ON n.embedding
OPTIONS {indexConfig: {`vector.dimensions`: 3072, `vector.similarity_function`: 'cosine'}}
WHERE n.layer = 'foundation';
```

---

## 5. MCP Tool Design

### 5.1 New Tools Required

#### 5.1.1 `foundation.list.v2` (Optimized)

**Purpose:** Load Foundation layer with field exclusion for token optimization

**Input Schema:**

```typescript
{
  excludeFields?: string[];  // default: ['properties', 'metadata', 'embedding', 'provenance']
  includeRelationships?: boolean;  // default: true
  status?: 'active' | 'archived' | 'deprecated';  // default: 'active'
}
```

**Output Schema:**

```typescript
{
  items: Array<{
    id: string;
    type: string;
    layer: 'foundation';
    title: string;
    summary?: string;
    description?: string; // Only for Value, Principle, Guardrail
    status: 'active' | 'archived' | 'deprecated';
    createdAt: string;
    updatedAt: string;
    // Excluded: properties, metadata, embedding, provenance
  }>;
  relationships: Array<{
    id: string;
    type: string;
    fromId: string;
    toId: string;
  }>;
  stats: {
    totalNodes: number;
    byType: Record<string, number>;
    tokenEstimate: number;
  }
}
```

**Implementation:**

```typescript
// In layerTools.ts
export const foundationListV2Tool: ToolDefinition = {
  name: 'odei.neo4j.foundation.list.v2',
  paramsSchema: z.object({
    excludeFields: z.array(z.string()).optional(),
    includeRelationships: z.boolean().default(true).optional(),
    status: z.enum(['active', 'archived', 'deprecated']).default('active').optional(),
  }),
  handler: async (params, ctx) => {
    const exclude = params.excludeFields || ['properties', 'metadata', 'embedding', 'provenance'];
    const query = `
      MATCH (n:OdeiNode)
      WHERE n.layer = 'foundation' AND n.status = $status
      RETURN n
      ORDER BY n.type, n.createdAt
    `;
    const result = await session.run(query, { status: params.status });
    const items = result.records.map((r) => {
      const node = toGraphNodeRecord(r.get('n'));
      // Remove excluded fields
      exclude.forEach((field) => delete node[field]);
      return node;
    });

    // Load relationships if requested
    let relationships = [];
    if (params.includeRelationships) {
      const relQuery = `
        MATCH (from:OdeiNode)-[r]->(to:OdeiNode)
        WHERE from.layer = 'foundation' AND to.layer = 'foundation'
          AND from.status = $status AND to.status = $status
        RETURN r.id AS id, type(r) AS type, from.id AS fromId, to.id AS toId
      `;
      const relResult = await session.run(relQuery, { status: params.status });
      relationships = relResult.records.map((r) => r.toObject());
    }

    // Estimate tokens
    const tokenEstimate = estimateTokens(JSON.stringify(items));

    return {
      items,
      relationships,
      stats: {
        totalNodes: items.length,
        byType: computeTypeCounts(items),
        tokenEstimate,
      },
    };
  },
};
```

#### 5.1.2 `vision.list.v2` (Optimized with Hierarchy)

**Purpose:** Load Vision layer with inline parent relationships

**Input Schema:**

```typescript
{
  excludeFields?: string[];  // default: ['properties', 'metadata', 'embedding']
  includeHierarchy?: boolean;  // default: true (inline parents)
  status?: 'active' | 'archived';  // default: 'active'
}
```

**Output Schema:**

```typescript
{
  items: Array<{
    id: string;
    type: 'vision' | 'business' | 'goal' | 'season';
    layer: 'vision';
    title: string;
    summary?: string;
    horizon?: string;
    deadline?: string;
    target?: string;
    status: 'active' | 'archived';
    parents?: Array<{
      // Inline hierarchy
      id: string;
      type: string;
      title: string;
      relationship: 'SUPPORTED_BY' | 'SERVES' | 'ALIGNS_WITH';
    }>;
  }>;
  stats: {
    totalNodes: number;
    byType: Record<string, number>;
    tokenEstimate: number;
  }
}
```

#### 5.1.3 `semantic.search.v1` (Unified Semantic Search)

**Purpose:** Vector similarity search across all layers

**Input Schema:**

```typescript
{
  query: string;  // Natural language query
  layers?: string[];  // default: all layers
  types?: string[];  // optional: filter by node types
  topK?: number;  // default: 10, max: 50
  minSimilarity?: number;  // default: 0.7, range: 0-1
  includeRelationships?: boolean;  // default: false
  status?: 'active' | 'archived';  // default: 'active'
}
```

**Output Schema:**

```typescript
{
  results: Array<{
    node: GraphNodeRecord;
    score: number;  // Similarity score 0-1
    relationships?: GraphRelationshipRecord[];  // If requested
  }>;
  stats: {
    totalResults: number;
    avgScore: number;
    tokenEstimate: number;
  };
  queryEmbedding?: number[];  // Optional: return for caching
}
```

**Implementation:**

```typescript
export const semanticSearchTool: ToolDefinition = {
  name: 'odei.neo4j.semantic.search.v1',
  paramsSchema: z.object({
    query: z.string().min(1),
    layers: z.array(z.enum(Layers)).optional(),
    types: z.array(z.string()).optional(),
    topK: z.number().int().min(1).max(50).default(10).optional(),
    minSimilarity: z.number().min(0).max(1).default(0.7).optional(),
    includeRelationships: z.boolean().default(false).optional(),
    status: z.enum(['active', 'archived']).default('active').optional(),
  }),
  handler: async (params, ctx) => {
    // Generate query embedding
    const queryEmbedding = await generateEmbedding(params.query, ctx);

    // Build layer filters
    const layerFilter = params.layers?.length ? `AND node.layer IN $layers` : '';
    const typeFilter = params.types?.length ? `AND node.type IN $types` : '';

    // Query vector index
    const query = `
      CALL db.index.vector.queryNodes(
        'all_embeddings',  // Unified index or layer-specific
        $topK,
        $queryEmbedding
      ) YIELD node, score
      WHERE node.status = $status
        ${layerFilter}
        ${typeFilter}
        AND score >= $minSimilarity
      RETURN node, score
      ORDER BY score DESC
    `;

    const result = await session.run(query, {
      topK: params.topK,
      queryEmbedding,
      status: params.status,
      layers: params.layers,
      types: params.types,
      minSimilarity: params.minSimilarity,
    });

    const results = await Promise.all(
      result.records.map(async (r) => {
        const node = toGraphNodeRecord(r.get('node'));
        const score = r.get('score');
        let relationships = [];

        if (params.includeRelationships) {
          relationships = await loadNodeRelationships(node.id, ctx);
        }

        return { node, score, relationships };
      })
    );

    return {
      results,
      stats: {
        totalResults: results.length,
        avgScore: results.reduce((sum, r) => sum + r.score, 0) / results.length,
        tokenEstimate: estimateTokens(JSON.stringify(results)),
      },
    };
  },
};
```

#### 5.1.4 `hybrid.search.v1` (Structural + Semantic)

**Purpose:** Combine relationship traversal with vector similarity

**Input Schema:**

```typescript
{
  query: string;  // Natural language query
  anchorNodeId?: string;  // Optional: start from specific node
  structuralWeight?: number;  // default: 0.6 (range: 0-1)
  semanticWeight?: number;  // default: 0.4 (range: 0-1)
  maxDepth?: number;  // default: 2 (relationship hops)
  topK?: number;  // default: 10
  minSimilarity?: number;  // default: 0.65 (lower than pure semantic)
}
```

**Output Schema:**

```typescript
{
  results: Array<{
    node: GraphNodeRecord;
    structuralScore: number; // 0-1
    semanticScore: number; // 0-1
    finalScore: number; // Weighted combination
    path?: Array<{
      // If anchorNodeId provided
      nodeId: string;
      relationship: string;
    }>;
  }>;
  stats: {
    totalResults: number;
    avgStructuralScore: number;
    avgSemanticScore: number;
    tokenEstimate: number;
  }
}
```

#### 5.1.5 `workload.assess.v1` (Enhanced with Calendar)

**Purpose:** Calculate current workload capacity from Tasks + Calendar

**Input Schema:**

```typescript
{
  startDate?: string;  // ISO 8601, default: today
  endDate?: string;    // ISO 8601, default: +7 days
  includeCalendar?: boolean;  // default: true
  includeTaskDetails?: boolean;  // default: false (just aggregates)
}
```

**Output Schema:**

```typescript
{
  capacity: {
    availableHours: number;      // Total work hours in period
    scheduledHours: number;      // Task effortHours + calendar events
    utilizationPercent: number;  // (scheduled / available) * 100
    zone: 'green' | 'yellow' | 'red';  // <75%, 75-85%, >85%
  };
  tasks: {
    total: number;
    byStatus: Record<string, number>;
    totalEffortHours: number;
    items?: Array<{  // If includeTaskDetails=true
      id: string;
      title: string;
      status: string;
      effortHours: number;
      dueDate: string;
    }>;
  };
  calendar: {
    total: number;
    totalHours: number;
    events?: Array<{  // Summary only, not full details
      title: string;
      start: string;
      duration: number;
    }>;
  };
  recommendations: string[];  // e.g., "At 88% capacity. Avoid new commitments without trade-offs."
}
```

---

## 6. Query Specifications

### 6.1 Session Start Query (Foundation + Vision)

**Composite query (run at session start):**

```typescript
async function loadSessionContext(ctx: ToolContext): Promise<SessionContext> {
  // Query 1: Foundation layer (optimized)
  const foundation = await ctx.tools['odei.neo4j.foundation.list.v2']({
    excludeFields: ['properties', 'metadata', 'embedding', 'provenance'],
    includeRelationships: true,
    status: 'active',
  });

  // Query 2: Vision layer (with hierarchy)
  const vision = await ctx.tools['odei.neo4j.vision.list.v2']({
    excludeFields: ['properties', 'metadata', 'embedding'],
    includeHierarchy: true,
    status: 'active',
  });

  // Query 3: Workload assessment
  const workload = await ctx.tools['odei.neo4j.workload.assess.v1']({
    startDate: new Date().toISOString(),
    endDate: addDays(new Date(), 7).toISOString(),
    includeCalendar: true,
    includeTaskDetails: false, // Just aggregates
  });

  return {
    foundation: foundation.items,
    foundationRelationships: foundation.relationships,
    vision: vision.items,
    workload: workload.capacity,
    stats: {
      totalNodes: foundation.stats.totalNodes + vision.stats.totalNodes,
      totalTokens: foundation.stats.tokenEstimate + vision.stats.tokenEstimate + 2000, // +2K for workload
      loadTime: Date.now() - startTime,
    },
  };
}
```

**Expected performance:**

- **Latency:** <3 seconds (parallel queries)
- **Tokens:** ~25K (15K foundation + 8K vision + 2K workload)
- **Network:** 3 tool calls (can be parallelized in MCP)

### 6.2 Mid-Conversation Retrieval Query

**Example: "What strategies support my Q1 revenue goal?"**

```typescript
// Step 1: Find goal node
const goalSearch = await ctx.tools['odei.neo4j.semantic.search.v1']({
  query: 'Q1 revenue goal',
  layers: ['vision'],
  types: ['goal'],
  topK: 1,
  minSimilarity: 0.8,
});

if (!goalSearch.results.length) {
  throw new Error('No matching goal found');
}

const goalId = goalSearch.results[0].node.id;

// Step 2: Hybrid search for strategies
const strategies = await ctx.tools['odei.neo4j.hybrid.search.v1']({
  query: 'revenue growth strategies',
  anchorNodeId: goalId,
  structuralWeight: 0.6, // Prioritize linked strategies
  semanticWeight: 0.4, // But also find semantically similar
  maxDepth: 2,
  topK: 10,
  minSimilarity: 0.65,
});

return {
  goal: goalSearch.results[0].node,
  strategies: strategies.results,
};
```

**Token budget:** ~8-12K (goal + 10 strategies with relationships)

### 6.3 Decision Precedent Query

**Example: "Find similar decisions with high ROI"**

```cypher
// Hybrid search: semantic similarity + ROI filter
WITH $queryEmbedding AS qEmbed, $minROI AS roi
CALL db.index.vector.queryNodes('execution_embeddings', 20, qEmbed)
YIELD node AS d, score
WHERE d.type = 'decision'
  AND d.status = 'active'
  AND score >= 0.6
  AND d.roi.total >= roi
OPTIONAL MATCH (d)-[:LEADS_TO]->(outcome:Task|Metric)
WHERE outcome.status IN ['done', 'active']
RETURN
  d,
  score,
  collect({id: outcome.id, type: outcome.type, title: outcome.title, status: outcome.status}) AS outcomes
ORDER BY d.roi.total DESC, score DESC
LIMIT 10
```

**Token budget:** ~10-15K (10 decisions + outcomes)

---

## 7. Performance Targets

### 7.1 Latency Targets

| Operation                    | Current | Target | P99 | Notes                          |
| ---------------------------- | ------- | ------ | --- | ------------------------------ |
| Session start (full context) | —       | <5s    | <8s | Foundation + Vision + Workload |
| Foundation.list.v2           | —       | <2s    | <3s | 45-80 nodes, optimized         |
| Vision.list.v2               | —       | <1s    | <2s | 0-25 nodes                     |
| Semantic search (topK=10)    | —       | <1.5s  | <3s | Vector query + node hydration  |
| Hybrid search (topK=10)      | —       | <2.5s  | <4s | Structural + semantic combine  |
| Workload.assess.v1           | —       | <2s    | <3s | Neo4j tasks + Calendar API     |

**Optimization strategies:**

- **Parallel queries:** Foundation + Vision + Workload can run simultaneously
- **Index warming:** Keep vector indexes in memory (Neo4j config)
- **Query result caching:** Client-side cache for session duration
- **Batch embeddings:** Generate embeddings async, don't block reads

### 7.2 Token Usage Targets

| Context Component   | Current  | Target   | Notes                        |
| ------------------- | -------- | -------- | ---------------------------- |
| System Prompt       | 30K      | 30K      | CLAUDE.md (unchanged)        |
| MCP Tools           | 15K      | 15K      | Tool schemas (unchanged)     |
| **Session Context** | **44K**  | **25K**  | **Optimized (see below)**    |
| - Foundation        | 44K      | 15K      | Exclude heavy fields         |
| - Vision            | 0K       | 8K       | Include hierarchy inline     |
| - Workload          | 0K       | 2K       | Aggregated view              |
| Conversation        | 111K     | 130K     | Increased from optimizations |
| **Total**           | **200K** | **200K** | Within budget                |

**Token optimization techniques:**

1. **Field exclusion:** Remove `properties`, `metadata`, `embedding`, `provenance` from list results
2. **Inline relationships:** Avoid separate relationship queries (1 query vs 2)
3. **Type-specific descriptions:** Only include `description` for Value/Principle/Guardrail
4. **Aggregated metrics:** Don't load individual Track nodes, only summaries
5. **Lazy loading:** Load Strategy/Tactics/Execution on demand, not at session start

### 7.3 Scaling Projections

**At 500 nodes (6 months):**

- Session context: 28K tokens (Foundation 17K + Vision 9K + Workload 2K)
- Semantic search: 8-12K tokens per query (topK=10)
- Latency: <6s session start, <2s semantic search

**At 5000 nodes (1 year):**

- Session context: 30K tokens (Foundation 20K + Vision 10K)
- Semantic search: 10-15K tokens per query (topK=15)
- Latency: <8s session start, <3s semantic search
- Requires: GDS algorithms for community detection, PageRank

---

## 8. Implementation Roadmap

### Phase 0: Fix Critical Issues (Week 1) — P0

**Status:** BLOCKING — Must fix before other work

**Tasks:**

1. **Generate real embeddings** (currently dummy zeros)
   - [ ] Run embedding generation script on existing 45 nodes
   - [ ] Verify embeddings in Neo4j browser (`MATCH (n:OdeiNode) RETURN n.embedding LIMIT 1`)
   - [ ] Test semantic search returns non-zero scores

2. **Create vector indexes** (missing from constraints.cypher)
   - [ ] Add vector index definitions to `/Users/ai/ODEI/servers/odei-neo4j/src/cypher/constraints.cypher`
   - [ ] Run setup script: `npm run odei:setup` or `node dist/setup.js`
   - [ ] Verify indexes: `SHOW INDEXES WHERE type = 'VECTOR'`

3. **Fix dimension mismatch** (config says 3072, dummy embeddings are zeros)
   - [ ] Verify `EMBEDDING_DIMENSIONS=3072` in `.claude/mcp.json`
   - [ ] Regenerate all embeddings with correct dimensions
   - [ ] Update embeddingManager.ts if needed

**Success Criteria:**

- ✅ All 45 nodes have real embeddings (non-zero vectors)
- ✅ Vector indexes created and populated (state: ONLINE)
- ✅ Semantic search returns meaningful results with scores >0

**Estimated Time:** 2-3 days

---

### Phase 1: Optimize Foundation Loading (Week 2-3) — P1

**Status:** HIGH PRIORITY — Fixes token overflow issue

**Tasks:**

1. **Implement foundation.list.v2 tool**
   - [ ] Add `excludeFields` parameter to layerTools.ts
   - [ ] Implement field exclusion in toGraphNodeRecord()
   - [ ] Add token estimation to response
   - [ ] Write tests for field exclusion

2. **Implement vision.list.v2 tool**
   - [ ] Add inline parent relationships query
   - [ ] Implement hierarchy construction
   - [ ] Optimize for token usage
   - [ ] Write tests for hierarchy loading

3. **Update Discuss agent to use v2 tools**
   - [ ] Modify session start protocol in CLAUDE.md
   - [ ] Update tool calls in agent workspace
   - [ ] Test session start latency
   - [ ] Verify token reduction (44K → 25K)

4. **Add workload.assess.v1 tool**
   - [ ] Query Task nodes for effortHours
   - [ ] Integrate Apple Calendar API
   - [ ] Calculate capacity percentages
   - [ ] Generate zone and recommendations

**Success Criteria:**

- ✅ Session start context loads in <5s
- ✅ Token usage reduced from 44K to ~25K
- ✅ All required context still available (no missing data)
- ✅ Discuss agent operates correctly with optimized tools

**Estimated Time:** 1-2 weeks

---

### Phase 2: Implement Hybrid Retrieval (Week 4-6) — P2

**Status:** MEDIUM PRIORITY — Enables scaling to 500+ nodes

**Tasks:**

1. **Implement semantic.search.v1 tool**
   - [ ] Add unified semantic search across layers
   - [ ] Support layer/type filtering
   - [ ] Implement minSimilarity threshold
   - [ ] Add relationship loading (optional)
   - [ ] Write tests for various query patterns

2. **Implement hybrid.search.v1 tool**
   - [ ] Combine structural (Cypher) + semantic (vector) queries
   - [ ] Implement weighted scoring (structural 0.6, semantic 0.4)
   - [ ] Add path traversal for anchorNodeId
   - [ ] Optimize for performance (<2.5s latency)
   - [ ] Write tests for hybrid scoring

3. **Add mid-conversation retrieval patterns**
   - [ ] Document common query patterns (strategy search, decision precedent, etc.)
   - [ ] Implement query templates in discuss-neo4j MCP
   - [ ] Add caching layer for repeated queries
   - [ ] Test token budgets for various query types

4. **Update agent workspaces**
   - [ ] Add new tools to Discuss agent allowlist
   - [ ] Update Decisions agent with decision precedent queries
   - [ ] Update Execute agent with task-based retrieval
   - [ ] Document retrieval patterns in agent CLAUDE.md files

**Success Criteria:**

- ✅ Semantic search returns relevant results with scores >0.7
- ✅ Hybrid search outperforms pure semantic (measured by user feedback)
- ✅ Mid-conversation retrieval uses <15K tokens per query
- ✅ All agents successfully use new retrieval tools

**Estimated Time:** 2-3 weeks

---

### Phase 3: Add GDS Algorithms (Month 3-4) — P3

**Status:** LOW PRIORITY — Optimization for 1000+ nodes

**Tasks:**

1. **Install Neo4j GDS plugin**
   - [ ] Add GDS to Neo4j Docker config or Enterprise installation
   - [ ] Verify GDS procedures available: `CALL gds.list()`
   - [ ] Test GDS on small dataset

2. **Implement graph algorithm tools**
   - [ ] `graph.pagerank.v1` — Find influential nodes
   - [ ] `graph.communities.v1` — Detect node clusters (Louvain)
   - [ ] `graph.betweenness.v1` — Find bridging nodes
   - [ ] `graph.shortest_path.v1` — Alignment path finding
   - [ ] Write tests for each algorithm

3. **Add algorithm-based retrieval patterns**
   - [ ] "What are the most influential Values?" (PageRank)
   - [ ] "What goal communities exist?" (Community detection)
   - [ ] "How does Task X align with Value Y?" (Shortest path)
   - [ ] Document patterns in agent guides

4. **Performance optimization**
   - [ ] Create GDS projections for common queries
   - [ ] Add caching for algorithm results (TTL: 1 hour)
   - [ ] Benchmark latency at 1000 nodes
   - [ ] Optimize slow queries

**Success Criteria:**

- ✅ GDS algorithms return meaningful insights
- ✅ PageRank identifies key constitutional nodes correctly
- ✅ Community detection reveals goal clusters
- ✅ Latency <5s for GDS queries at 1000 nodes

**Estimated Time:** 3-4 weeks

---

### Phase 4: Topic Extraction & Advanced Patterns (Month 5-6) — P4

**Status:** FUTURE — Advanced features for 2000+ nodes

**Tasks:**

1. **Implement topic extraction**
   - [ ] Cluster node embeddings (k-means or HDBSCAN)
   - [ ] Extract representative keywords per cluster
   - [ ] Add topic tags to nodes
   - [ ] Build topic-based navigation

2. **Add temporal evolution tracking**
   - [ ] Track how Values/Principles change over time
   - [ ] Detect constitutional drift patterns
   - [ ] Generate evolution narratives
   - [ ] Alert on significant changes

3. **Implement pattern detection**
   - [ ] Recurring goal failure patterns
   - [ ] High-ROI decision templates
   - [ ] Successful strategy archetypes
   - [ ] Feed patterns to Mind layer

4. **Advanced retrieval features**
   - [ ] Multi-hop reasoning (Why does Goal X serve Value Y?)
   - [ ] Counterfactual queries (What if we removed Principle X?)
   - [ ] Analogy search (Goals similar to past successes)
   - [ ] Contradiction detection (automated)

**Success Criteria:**

- ✅ Topic extraction reveals meaningful clusters
- ✅ Temporal evolution tracks constitutional changes accurately
- ✅ Pattern detection surfaces actionable insights
- ✅ Advanced retrieval enables sophisticated queries

**Estimated Time:** 4-6 weeks

---

## 9. Test Plan & Success Metrics

### 9.1 Unit Tests

**Embedding generation:**

- [ ] Test embedding generation for all node types
- [ ] Verify embedding dimensions (3072)
- [ ] Test skip logic (no meaningful field changes)
- [ ] Test retry logic (exponential backoff)

**Field exclusion:**

- [ ] Test foundation.list.v2 excludes specified fields
- [ ] Test vision.list.v2 includes parent hierarchy
- [ ] Verify token reduction (measure with tiktoken)

**Search tools:**

- [ ] Test semantic search with various similarity thresholds
- [ ] Test hybrid search scoring (structural + semantic)
- [ ] Verify result ordering (score DESC)
- [ ] Test layer/type filtering

**Workload assessment:**

- [ ] Test capacity calculation (Task effortHours + Calendar events)
- [ ] Test zone thresholds (green <75%, yellow 75-85%, red >85%)
- [ ] Test recommendations generation

### 9.2 Integration Tests

**Session start flow:**

- [ ] Test parallel loading (Foundation + Vision + Workload)
- [ ] Verify all required context loaded
- [ ] Measure total latency (<5s target)
- [ ] Measure total tokens (~25K target)

**Mid-conversation retrieval:**

- [ ] Test goal-driven strategy search
- [ ] Test decision precedent search
- [ ] Test insight semantic search
- [ ] Verify token budgets (<15K per query)

**Agent workflows:**

- [ ] Test Discuss agent session start protocol
- [ ] Test Decisions agent decision precedent lookup
- [ ] Test Execute agent workload assessment
- [ ] Test Mind agent insight search

### 9.3 Performance Benchmarks

**Latency benchmarks (at various graph sizes):**

| Graph Size | Foundation.list | Vision.list | Semantic Search | Hybrid Search | Session Start |
| ---------- | --------------- | ----------- | --------------- | ------------- | ------------- |
| 45 nodes   | <1s             | <0.5s       | <1s             | <1.5s         | <3s           |
| 500 nodes  | <2s             | <1s         | <1.5s           | <2.5s         | <5s           |
| 1000 nodes | <2.5s           | <1s         | <2s             | <3s           | <6s           |
| 5000 nodes | <3s             | <1.5s       | <3s             | <4s           | <8s           |

**Token benchmarks:**

| Context Component | 45 Nodes | 500 Nodes | 1000 Nodes | 5000 Nodes |
| ----------------- | -------- | --------- | ---------- | ---------- |
| Foundation        | 15K      | 17K       | 18K        | 20K        |
| Vision            | 2K       | 9K        | 10K        | 10K        |
| Workload          | 2K       | 2K        | 2K         | 2K         |
| **Total Session** | **19K**  | **28K**   | **30K**    | **32K**    |
| **Conversation**  | **166K** | **157K**  | **155K**   | **153K**   |

### 9.4 Quality Metrics

**Retrieval relevance:**

- **Semantic search precision@10:** >80% (manually labeled test set)
- **Hybrid search precision@10:** >85% (should outperform pure semantic)
- **False positive rate:** <10% (irrelevant results returned)

**Embedding quality:**

- **Cosine similarity for duplicates:** >0.9 (same content should cluster)
- **Cosine similarity for unrelated:** <0.4 (different concepts should separate)
- **Cluster coherence:** Silhouette score >0.5

**Agent satisfaction:**

- **Session start success rate:** >95% (all required context loaded)
- **Query satisfaction:** >90% (user feedback: relevant results)
- **Context completeness:** >98% (no missing critical nodes)

### 9.5 Regression Tests

**After each deployment:**

1. Run all unit tests (must pass 100%)
2. Run integration tests (must pass >95%)
3. Benchmark latency (must meet targets)
4. Check token usage (must stay within budget)
5. Verify embedding quality (spot check 10 random nodes)
6. Test session start flow (all agents)
7. Validate retrieval relevance (sample queries)

---

## 10. Migration Path

### 10.1 Current State → Phase 0 (Fix Embeddings)

**Migration steps:**

1. **Backup existing graph:**

   ```bash
   neo4j-admin database dump memory --to-path=/backups/pre-embedding-fix
   ```

2. **Generate real embeddings for existing nodes:**

   ```bash
   node /Users/ai/ODEI/servers/odei-neo4j/scripts/regenerate-embeddings.js
   ```

3. **Create vector indexes:**

   ```bash
   # Run setup script (idempotent)
   cd /Users/ai/ODEI/servers/odei-neo4j
   npm run setup
   ```

4. **Verify embeddings:**

   ```cypher
   // Check embedding dimensions
   MATCH (n:OdeiNode)
   WHERE n.embedding IS NOT NULL
   RETURN n.type, size(n.embedding) AS dimensions, count(*) AS count;

   // Should return: dimensions = 3072 for all types
   ```

5. **Test semantic search:**

   ```typescript
   // Via MCP tool
   const result = await discussNeo4j.tools['discuss_semantic_search']({
     query: 'solo entrepreneurship values',
     topK: 5,
   });

   // Should return: 5 results with scores >0
   ```

**Rollback plan:**

- If embeddings fail → restore from backup
- If indexes fail → drop indexes, fix config, retry
- No data loss risk (embeddings are additive)

---

### 10.2 Phase 0 → Phase 1 (Optimize Foundation Loading)

**Migration steps:**

1. **Deploy new tools (backward compatible):**

   ```bash
   cd /Users/ai/ODEI/servers/odei-neo4j
   npm run build
   # Restart MCP servers (or reload in Claude Code)
   ```

2. **Test v2 tools in parallel with v1:**

   ```typescript
   // Compare outputs
   const v1 = await odeiNeo4j.tools['odei.neo4j.foundation.list.v1']({});
   const v2 = await odeiNeo4j.tools['odei.neo4j.foundation.list.v2']({
     excludeFields: ['properties', 'metadata', 'embedding', 'provenance'],
   });

   // Verify: v2.items has same nodes as v1.items, but fewer fields
   // Verify: v2.stats.tokenEstimate < v1 token count (measure with tiktoken)
   ```

3. **Update agent configurations:**

   ```bash
   # Update agents/discuss/.claude/CLAUDE.md
   # Change: odei.neo4j.foundation.list.v1 → foundation.list.v2
   ```

4. **Monitor token usage in production:**

   ```typescript
   // Add telemetry
   ctx.logger.info({
     tool: 'foundation.list.v2',
     tokenEstimate: result.stats.tokenEstimate,
     nodeCount: result.items.length,
   });
   ```

5. **Deprecate v1 tools (after 1 month):**
   - Remove from tool registry
   - Update documentation
   - Archive old tool code

**Rollback plan:**

- Keep v1 tools active during migration
- If v2 fails → agents fall back to v1
- No data changes (read-only optimization)

---

### 10.3 Phase 1 → Phase 2 (Hybrid Retrieval)

**Migration steps:**

1. **Deploy new retrieval tools:**

   ```bash
   npm run build
   # Restart MCP servers
   ```

2. **Add to agent allowlists:**

   ```json
   // agents/discuss/.claude/mcp.json
   {
     "toolAllowList": [
       "odei.neo4j.semantic.search.v1",
       "odei.neo4j.hybrid.search.v1"
       // ... existing tools
     ]
   }
   ```

3. **Update agent prompts with retrieval patterns:**

   ```markdown
   ## Mid-Conversation Retrieval Patterns

   **Pattern: Find strategies for goal**

   1. Search for goal: semantic.search.v1(query: "goal name", layers: ["vision"])
   2. Find strategies: hybrid.search.v1(query: "strategy query", anchorNodeId: goalId)

   **Pattern: Find decision precedent**

   1. Semantic search: semantic.search.v1(query: "decision context", layers: ["execution"], types: ["decision"])
   2. Filter by ROI: client-side filter for roi.total >= threshold
   ```

4. **Test in production with monitoring:**
   - Log all retrieval queries
   - Track latency and token usage
   - Collect user feedback on relevance

5. **Iterate based on feedback:**
   - Adjust similarity thresholds
   - Tune structural/semantic weights
   - Optimize slow queries

**Rollback plan:**

- New tools are additive (don't break existing workflows)
- Agents can continue using foundation.list/vision.list
- Remove from allowlist if issues occur

---

### 10.4 Phase 2 → Phase 3 (GDS Algorithms)

**Migration steps:**

1. **Install Neo4j GDS plugin:**

   ```bash
   # Docker: Add to NEO4J_PLUGINS env var
   NEO4J_PLUGINS='["graph-data-science"]'

   # Or install manually
   neo4j-admin install-plugin gds
   neo4j restart
   ```

2. **Verify GDS available:**

   ```cypher
   CALL gds.list() YIELD name;
   // Should return: gds.pageRank, gds.louvain, etc.
   ```

3. **Create GDS projections:**

   ```cypher
   // Constitutional graph projection
   CALL gds.graph.project(
     'constitutional-graph',
     'OdeiNode',
     {
       SUPPORTED_BY: {orientation: 'NATURAL'},
       ENFORCED_BY: {orientation: 'NATURAL'},
       ALIGNS_WITH: {orientation: 'NATURAL'},
       SERVES: {orientation: 'NATURAL'}
     },
     {nodeProperties: ['layer', 'type', 'status']}
   );
   ```

4. **Deploy GDS tools:**

   ```bash
   npm run build
   # Add to agent allowlists
   ```

5. **Test algorithms on production graph:**

   ```cypher
   // PageRank (find influential nodes)
   CALL gds.pageRank.stream('constitutional-graph')
   YIELD nodeId, score
   RETURN gds.util.asNode(nodeId).title AS title, score
   ORDER BY score DESC
   LIMIT 10;
   ```

6. **Monitor performance:**
   - Track GDS query latency
   - Measure memory usage
   - Optimize projections if needed

**Rollback plan:**

- GDS is optional (doesn't affect core retrieval)
- Disable GDS tools if performance issues
- Drop projections if memory pressure

---

## 11. Open Questions & Future Considerations

### 11.1 Open Questions

**Q1: Should we use separate vector indexes per layer or one unified index?**

**Option A: Separate indexes (current design)**

- Pros: Faster queries (smaller index), layer-specific tuning
- Cons: More indexes to manage, can't cross-layer search easily

**Option B: Unified index with layer metadata filter**

- Pros: Simpler, enables cross-layer queries
- Cons: Slower queries (larger index), no layer-specific optimization

**Recommendation:** Start with separate indexes (Phase 0-2), consider unified in Phase 3 if cross-layer queries are common.

---

**Q2: Should we downsample embeddings from 3072 to 1536 dimensions?**

**Trade-off:**

- 3072: Higher accuracy, 2× cost, 2× storage
- 1536: Lower accuracy, 1× cost, 1× storage

**Cost analysis at 5000 nodes:**

- 3072: $2-3 initial + $0.05/month
- 1536: $1-1.5 initial + $0.025/month

**Recommendation:** Use 3072 (current config). Cost delta is negligible ($0.02/month), accuracy matters for constitutional decisions.

---

**Q3: How often should we recompute embeddings to detect drift?**

**Options:**

- Never (only on create/update) — Risk: embedding model updates not reflected
- Weekly batch — Balance: catch major drifts without constant recomputation
- Monthly batch — Conservative: minimize API costs

**Recommendation:** Weekly batch for P0-P1 nodes (Foundation, Vision), monthly for P2-P4. Monitor drift metrics (cosine similarity change >0.1 = significant drift).

---

**Q4: Should we cache query embeddings client-side?**

**Pros:**

- Avoid regenerating embeddings for repeated queries
- Reduce OpenAI API costs
- Faster retrieval (skip embedding generation)

**Cons:**

- Cache invalidation complexity
- Client-side storage requirements
- Stale embeddings if model updates

**Recommendation:** Implement in Phase 2 with 15-minute TTL. Cache key: hash(query + model + dimensions).

---

### 11.2 Future Considerations

**Multi-modal embeddings:**

- Current: Text-only (OpenAI text-embedding-3-large)
- Future: Images, audio, video (OpenAI CLIP, Whisper)
- Use case: Memory Atlas visualizations, voice memos, design artifacts

**Embedding fine-tuning:**

- Current: Generic OpenAI embeddings
- Future: Fine-tune on ODEI-specific corpus
- Use case: Better understanding of constitutional language, personal terminology

**Graph embeddings (Node2Vec, GraphSAGE):**

- Current: Text embeddings only (ignore graph structure)
- Future: Combine text + graph structure embeddings
- Use case: Better capture of relational context (Value → Principle → Guardrail chains)

**Federated search across agents:**

- Current: Each agent has separate MCP server
- Future: Cross-agent search (Discuss queries Decisions data, etc.)
- Challenge: Access control, provenance tracking

**Real-time embedding updates:**

- Current: Async embedding generation (write succeeds before embedding done)
- Future: Stream embeddings as they're generated
- Use case: Immediate semantic search on newly created nodes

**Vector index optimization:**

- Current: Default Neo4j vector index settings
- Future: Tune HNSW parameters (ef_construction, M) for performance
- Benchmark: Latency vs accuracy trade-offs at 5000+ nodes

---

## 12. Appendix

### 12.1 Glossary

- **Constitutional graph:** Foundation (Values, Principles, Guardrails) + Vision (Goals, Visions)
- **Embedding:** Vector representation of text (3072 dimensions for text-embedding-3-large)
- **Vector index:** Neo4j index for fast similarity search on embeddings
- **Semantic search:** Find nodes by meaning (vector similarity)
- **Structural search:** Find nodes by relationships (Cypher graph traversal)
- **Hybrid search:** Combine semantic + structural with weighted scoring
- **Session context:** Foundation + Vision + Workload loaded at session start
- **Token budget:** Max tokens in Claude's context window (200K total)

### 12.2 References

**Neo4j Documentation:**

- [Vector Indexes](https://neo4j.com/docs/cypher-manual/current/indexes-for-vector-search/)
- [GenAI Plugin](https://neo4j.com/labs/genai-ecosystem/openai/)
- [GDS Library](https://neo4j.com/docs/graph-data-science/current/)

**OpenAI Embeddings:**

- [text-embedding-3-large](https://platform.openai.com/docs/guides/embeddings)
- [Embedding Best Practices](https://platform.openai.com/docs/guides/embeddings/use-cases)

**ODEI Documentation:**

- `/Users/ai/ODEI/.claude/CLAUDE.md` — Workspace architecture
- `/Users/ai/ODEI/agents/discuss/.claude/CLAUDE.md` — Discuss agent protocol
- `/Users/ai/ODEI/servers/odei-neo4j/` — MCP server implementation

### 12.3 Cypher Query Templates

**Template: Load foundation with relationships**

```cypher
MATCH (n:OdeiNode)
WHERE n.layer = 'foundation' AND n.status = 'active'
WITH collect(n) AS nodes
UNWIND nodes AS n
OPTIONAL MATCH (n)-[r]->(m)
WHERE m IN nodes
RETURN
  n.id, n.type, n.title, n.summary, n.description,
  collect({to: m.id, type: type(r)}) AS relationships
```

**Template: Goal ladder (hierarchical goals)**

```cypher
MATCH path = (goal:Goal {id: $goalId})-[:SUPPORTED_BY*0..5]->(parent:Goal|Vision)
WHERE all(node IN nodes(path) WHERE node.status = 'active')
RETURN
  [node IN nodes(path) | {id: node.id, type: node.type, title: node.title}] AS ladder,
  length(path) AS depth
ORDER BY depth
LIMIT 1
```

**Template: Find strategies aligned with goal**

```cypher
MATCH (goal:Goal {id: $goalId})
MATCH (strategy:Strategy)-[:ALIGNS_WITH|SERVES*1..2]->(goal)
WHERE strategy.status = 'active'
OPTIONAL MATCH (strategy)-[:SUPPORTED_BY]->(initiative:Initiative)
WHERE initiative.status = 'active'
RETURN strategy, collect(initiative) AS initiatives
ORDER BY strategy.updatedAt DESC
```

**Template: Decision precedent with ROI**

```cypher
MATCH (d:Decision)
WHERE d.status = 'active'
  AND d.roi.total >= $minROI
  AND d.decidedAt >= $startDate
OPTIONAL MATCH (d)-[:LEADS_TO]->(outcome:Task)
WHERE outcome.status = 'done'
RETURN d, collect(outcome) AS outcomes
ORDER BY d.roi.total DESC
LIMIT $limit
```

### 12.4 MCP Tool Catalog

**Current tools (odei-neo4j):**

- Layer tools: `foundation.list.v1`, `vision.list.v1`, `strategy.list.v1`, etc.
- Node type tools: `value.create.v1`, `principle.update.v1`, etc. (64 total)
- Graph tools: `graph.getAll.v1`, `schema.inventory.v1`, `relationship.create.v1`
- Node operations: `node.update.v1`, `node.delete.v1`
- Utility: `context.usage.v1`, `backup.restore.v1`

**New tools (proposed):**

- `foundation.list.v2` — Optimized foundation loading with field exclusion
- `vision.list.v2` — Vision loading with inline hierarchy
- `semantic.search.v1` — Unified semantic search across layers
- `hybrid.search.v1` — Combine structural + semantic retrieval
- `workload.assess.v1` — Task + Calendar capacity calculation
- `graph.pagerank.v1` — GDS PageRank for influential nodes
- `graph.communities.v1` — GDS community detection
- `graph.betweenness.v1` — GDS betweenness centrality

**Agent-specific tools (discuss-neo4j):**

- `discuss_semantic_search` — Semantic search wrapper
- `discuss_hybrid_search` — Hybrid search wrapper
- `discuss_goal_ladder` — Goal hierarchy traversal
- `discuss_workload_assess` — Workload capacity + calendar
- `discuss_graph_pagerank` — PageRank wrapper
- `discuss_graph_communities` — Community detection wrapper
- Additional: `discuss_expand_path`, `discuss_subgraph_nodes`, `discuss_concept_bridge`, etc.

---

**Document Status:** Complete
**Next Steps:** Review → Approve → Begin Phase 0 implementation
**Owner:** ODEI Core Team
**Last Updated:** 2025-11-04
