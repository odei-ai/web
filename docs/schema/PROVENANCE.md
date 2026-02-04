# ODEI Provenance & RBAC System

**Version:** 1.0
**Generated:** 2025-12-25

---

## Table of Contents

1. [What is Provenance?](#what-is-provenance)
2. [Provenance Schema](#provenance-schema)
3. [Module-Based RBAC](#module-based-rbac)
4. [Actor Types & Authority](#actor-types--authority)
5. [Confidence Scoring](#confidence-scoring)
6. [Common Patterns](#common-patterns)
7. [Audit & Traceability](#audit--traceability)

---

## What is Provenance?

**Provenance** tracks the origin and context of every write operation in the ODEI graph. Every node and relationship creation requires a provenance block that answers:

- **WHO** made this change? (module + actor)
- **WHAT** was the source? (tool name, conversation ID, etc.)
- **HOW CERTAIN** are we? (confidence score 0.0-1.0)
- **WHY** was this done? (optional notes field)

This creates a complete audit trail and enables:
- **Conflict resolution** — Human edits override agent edits
- **Confidence-based filtering** — Show only high-confidence insights
- **Analytics** — Understand which modules create what content
- **Debugging** — Trace errors back to source operations
- **Regulatory compliance** — Immutable audit logs

---

## Provenance Schema

Every write operation requires this structure:

```typescript
{
  module: "discuss" | "plan" | "execute" | "mind",
  actor: "human" | "agent" | "joint",
  source: string,      // Tool name, conversation ID, script name
  confidence: number,  // 0.0-1.0 (how certain is this information?)
  notes?: string       // Optional context (max 500 chars)
}
```

### Field Descriptions

#### `module` (required)
Which operational module performed the write.

| Module    | Purpose                                    | Writable Layers           |
|-----------|-------------------------------------------|---------------------------|
| `discuss` | Constitutional dialogue, vision alignment | foundation, vision        |
| `plan`    | Strategic decomposition, OKR planning     | strategy, tactics         |
| `execute` | Task execution, time management           | tactics, execution, track |
| `mind`    | Pattern detection, insights, analytics    | track, mind               |

#### `actor` (required)
Who initiated this operation.

| Actor   | Authority Level | Description                                    |
|---------|----------------|------------------------------------------------|
| `human` | Highest        | Direct human input (highest trust)             |
| `joint` | High           | Human-approved agent work                      |
| `agent` | Medium         | Autonomous agent operation (needs validation)  |

#### `source` (required)
Identifier for the operation source. Examples:
- `"odei.neo4j.goal.create.v1"` — Tool name
- `"conv_abc123"` — Conversation ID
- `"script:weekly_review.js"` — Automated script
- `"import:notion_2025_12_25"` — Data import
- `"manual:web_ui"` — Direct web UI input

#### `confidence` (required)
Confidence score from 0.0 (uncertain) to 1.0 (certain).

| Range     | Interpretation               | Use Case                                  |
|-----------|------------------------------|-------------------------------------------|
| 0.9-1.0   | Very high confidence         | Human input, verified facts               |
| 0.7-0.89  | High confidence              | Agent with strong evidence                |
| 0.5-0.69  | Medium confidence            | Agent inference, needs validation         |
| 0.3-0.49  | Low confidence               | Speculative, experimental                 |
| 0.0-0.29  | Very low confidence          | Placeholder, draft, needs review          |

#### `notes` (optional)
Free-form context about this operation. Max 500 characters.

**Examples:**
- `"Created during Q4 planning session"`
- `"Imported from Notion database"`
- `"Inferred from completion pattern analysis"`
- `"User requested via Discuss agent chat"`

---

## Module-Based RBAC

### Permission Matrix

| Module    | Foundation | Vision | Strategy | Tactics | Execution | Track | Mind |
|-----------|-----------|--------|----------|---------|-----------|-------|------|
| `discuss` | ✅ Write   | ✅ Write | 🔍 Search | 🔍 Search | 📊 Summary | 📊 Summary | 📊 Summary |
| `plan`    | 🔍 Search | 🔍 Search | ✅ Write | ✅ Write | 🔍 Search | 📊 Summary | 📊 Summary |
| `execute` | 🔍 Search | 🔍 Search | 🔍 Search | ✅ Write | ✅ Write | ✅ Write | 📊 Summary |
| `mind`    | 🔍 Search | 🔍 Search | 🔍 Search | 🔍 Search | 🔍 Search | ✅ Write | ✅ Write |

**Legend:**
- ✅ **Write** — Full CRUD access (create, read, update, delete)
- 🔍 **Search** — Read-only via hybrid search
- 📊 **Summary** — Aggregated data only (e.g., workload assessment)

### Deny Lists

Even within allowed layers, modules cannot mutate specific node types:

```typescript
export const MODULE_NODE_DENY_LIST: Record<string, string[]> = {
  discuss: [],  // Full foundation/vision access
  plan: ['task', 'time_block', 'work_session', 'action'],
  execute: ['value', 'principle', 'guardrail'],
  mind: ['value', 'principle', 'guardrail'],
};
```

**Why deny lists?**
- **plan** should not create execution-level tasks (Execute owns that)
- **execute** should not modify constitutional values (Discuss owns that)
- **mind** is read-only observer, should not modify foundation

### Validation Flow

When a write operation arrives:

1. **Parse provenance** → Validate required fields (module, actor, source, confidence)
2. **Check layer permission** → Is `module` allowed to write to this `layer`?
3. **Check deny list** → Is `nodeType` explicitly denied for this `module`?
4. **Execute write** → If all checks pass, proceed to Neo4j

**Example rejection:**

```typescript
// Execute agent tries to create a Value (foundation layer)
{
  type: "value",
  data: { title: "New Value", ... },
  provenance: {
    module: "execute",  // ❌ execute cannot write foundation
    actor: "agent",
    source: "odei.neo4j.value.create.v1",
    confidence: 0.9
  }
}

// Guardian Error:
GuardianError(
  'GUARD_FORBIDDEN_OPERATION',
  'Module "execute" cannot mutate foundation layer',
  { module: 'execute', layer: 'foundation', nodeType: 'value' }
)
```

---

## Actor Types & Authority

### Human Actor

**Use when:**
- Direct user input via UI
- Manual edits in Discuss/Plan/Execute agents
- CLI commands run by human
- Data imported from trusted human sources

**Authority:**
- Highest priority in conflict resolution
- Can override agent-created content
- Sets precedent for future agent behavior

**Example:**
```typescript
{
  module: "discuss",
  actor: "human",
  source: "web_ui:goal_editor",
  confidence: 1.0,
  notes: "User manually created goal during planning session"
}
```

### Agent Actor

**Use when:**
- Autonomous agent operation
- Scheduled tasks and automations
- Pattern detection and inference
- Speculative or experimental content

**Authority:**
- Medium priority (can be overridden by human)
- Requires validation for critical operations
- Used for bulk operations and analysis

**Example:**
```typescript
{
  module: "mind",
  actor: "agent",
  source: "pattern_detection:weekly_cron",
  confidence: 0.75,
  notes: "Pattern detected from 23 task completion observations"
}
```

### Joint Actor

**Use when:**
- Human reviewed and approved agent work
- Collaborative human-AI editing
- Agent suggestions accepted by human
- Human-guided agent operations

**Authority:**
- High priority (human-validated agent work)
- Best of both: human judgment + agent automation
- Recommended for strategic decisions

**Example:**
```typescript
{
  module: "plan",
  actor: "joint",
  source: "conv_abc123",
  confidence: 0.95,
  notes: "Agent proposed OKR breakdown, user approved and refined"
}
```

---

## Confidence Scoring

### Calibration Guidelines

#### 1.0 — Absolute Certainty
- Human-entered facts
- System-generated timestamps
- Immutable historical records

**Example:**
```typescript
{
  type: "event",
  data: {
    title: "Aegis.im Exit",
    occurredAt: "2024-03-15T12:00:00Z"
  },
  provenance: {
    module: "mind",
    actor: "human",
    source: "manual:career_history_import",
    confidence: 1.0,
    notes: "Verified historical event"
  }
}
```

#### 0.9 — Very High Confidence
- Human input with possible minor errors
- Verified agent inference
- Trusted data imports

**Example:**
```typescript
{
  type: "goal",
  data: {
    title: "Build $500M Net Worth by 2035",
    deadline: "2035-01-01"
  },
  provenance: {
    module: "discuss",
    actor: "human",
    source: "conv_vision_session_20251225",
    confidence: 0.95,
    notes: "User stated goal during vision planning"
  }
}
```

#### 0.7-0.8 — High Confidence
- Agent inference with strong evidence
- Pattern detection with large sample size
- Validated agent recommendations

**Example:**
```typescript
{
  type: "insight",
  data: {
    title: "Evening strategic pivots fail 87% of time",
    confidence: 0.87
  },
  provenance: {
    module: "mind",
    actor: "agent",
    source: "pattern_analysis:decision_outcomes",
    confidence: 0.8,
    notes: "Based on 23 decision outcomes, 20 failures"
  }
}
```

#### 0.5-0.6 — Medium Confidence
- Speculative agent inference
- Limited data, needs validation
- Experimental features

**Example:**
```typescript
{
  type: "pattern",
  data: {
    title: "UI tasks finish 20% faster than estimated",
    confidence: 0.6
  },
  provenance: {
    module: "mind",
    actor: "agent",
    source: "velocity_analysis:limited_sample",
    confidence: 0.6,
    notes: "Only 5 data points, needs more observations"
  }
}
```

#### < 0.5 — Low Confidence
- Placeholders and drafts
- Highly speculative content
- Needs human review

**Example:**
```typescript
{
  type: "risk",
  data: {
    title: "Potential partnership execution risk",
    probability: 0.35
  },
  provenance: {
    module: "plan",
    actor: "agent",
    source: "risk_detection:experimental",
    confidence: 0.4,
    notes: "Speculative risk, flagged for human review"
  }
}
```

---

## Common Patterns

### Pattern 1: Human-Created Goal

```typescript
// User creates goal via Discuss agent
await neo4j.goal.create.v1({
  data: {
    title: "Launch Tipz CBS MVP",
    summary: "Ship minimum viable product for Tipz CBS by Q1 2026",
    horizon: "quarter",
    deadline: "2026-03-31"
  },
  provenance: {
    module: "discuss",
    actor: "human",
    source: "conv_vision_planning_2025_12_25",
    confidence: 1.0,
    notes: "User-defined goal during vision planning session"
  }
});
```

### Pattern 2: Agent-Generated Tasks from Project

```typescript
// Execute agent breaks down project into tasks
await neo4j.task.create.v1({
  data: {
    title: "Build agent panel UI component",
    summary: "Create React component for agent status panel",
    effortHours: 8,
    priority: "p1"
  },
  provenance: {
    module: "execute",
    actor: "agent",
    source: "project_breakdown:automated",
    confidence: 0.85,
    notes: "Auto-generated from project decomposition"
  }
});
```

### Pattern 3: Joint Pattern Detection

```typescript
// Mind agent proposes pattern, human reviews and accepts
await neo4j.pattern.create.v1({
  data: {
    title: "Solo+AI Delivers: Proven through 3 exits",
    summary: "Pattern of successful solo+AI ventures",
    pattern: "Solo founder + AI co-principal yields exits",
    confidence: 0.9
  },
  provenance: {
    module: "mind",
    actor: "joint",
    source: "pattern_review:human_approved",
    confidence: 0.95,
    notes: "Agent detected pattern, human validated and refined title"
  }
});
```

### Pattern 4: Bulk Import with Varying Confidence

```typescript
// Import historical data with different confidence levels
const importData = [
  {
    title: "Aegis.im Exit",
    occurredAt: "2024-03-15",
    confidence: 1.0,  // Verified historical fact
    notes: "Confirmed exit date from records"
  },
  {
    title: "Started consulting work",
    occurredAt: "2023-06-01",
    confidence: 0.7,  // Approximate date
    notes: "Estimated start date, exact date unknown"
  }
];

for (const item of importData) {
  await neo4j.event.create.v1({
    data: item,
    provenance: {
      module: "mind",
      actor: "human",
      source: "import:career_history_2025_12_25",
      confidence: item.confidence,
      notes: item.notes
    }
  });
}
```

---

## Audit & Traceability

### Query All Operations by Module

```cypher
// Find all nodes created by Execute agent
MATCH (n)
WHERE n.provenance.module = 'execute'
RETURN n.type AS node_type,
       n.title AS title,
       n.createdAt AS created,
       n.provenance.actor AS actor,
       n.provenance.source AS source
ORDER BY n.createdAt DESC
LIMIT 50
```

### Query Low-Confidence Content

```cypher
// Find nodes needing human review (low confidence)
MATCH (n)
WHERE n.provenance.confidence < 0.5
  AND n.status = 'draft'
RETURN n.type AS type,
       n.title AS title,
       n.provenance.confidence AS confidence,
       n.provenance.notes AS context
ORDER BY n.provenance.confidence ASC
```

### Conflict Resolution: Human vs Agent

```cypher
// Find nodes where human overrode agent work
MATCH (n)
WHERE n.provenance.actor = 'human'
  AND n.updatedAt > n.createdAt
OPTIONAL MATCH (n)<-[prev_edit]-(agent_version)
WHERE agent_version.provenance.actor = 'agent'
RETURN n.title AS node,
       n.provenance.source AS human_source,
       agent_version.provenance.source AS previous_agent_source,
       n.updatedAt AS override_timestamp
```

### Trace Agent Decisions

```cypher
// Trace agent decision chain
MATCH (d:Decision)
WHERE d.provenance.actor = 'agent'
OPTIONAL MATCH (d)-[:SUPPORTED_BY]->(e:Evidence)
OPTIONAL MATCH (ins:Insight)-[:TRIGGERS]->(d)
RETURN d.title AS decision,
       d.provenance.source AS agent_source,
       d.roi.total AS roi_score,
       collect(e.title) AS evidence,
       collect(ins.title) AS triggered_by
ORDER BY d.decidedAt DESC
```

### Module Activity Report

```cypher
// Activity breakdown by module (last 30 days)
MATCH (n)
WHERE n.createdAt >= datetime() - duration('P30D')
RETURN n.provenance.module AS module,
       count(n) AS nodes_created,
       avg(n.provenance.confidence) AS avg_confidence,
       collect(DISTINCT n.type) AS node_types
ORDER BY nodes_created DESC
```

---

## Best Practices

### ✅ DO

1. **Always provide provenance** — Required for all writes
2. **Use appropriate actor type** — Human for direct input, agent for automation, joint for reviewed work
3. **Calibrate confidence accurately** — Don't overstate certainty
4. **Include meaningful notes** — Help future debugging
5. **Use specific sources** — `"conv_abc123"` not `"conversation"`

### ❌ DON'T

1. **Don't use confidence = 1.0 for agent work** — Unless verified
2. **Don't omit provenance** — Will be rejected by Guardian
3. **Don't use generic sources** — `"agent"` is too vague
4. **Don't modify provenance after creation** — It's immutable
5. **Don't bypass RBAC** — Respect layer boundaries

---

## Future Enhancements

### Planned Features

1. **Provenance Signatures** — Cryptographic signing for audit compliance
2. **Provenance Chains** — Track edit history across updates
3. **Automated Confidence Decay** — Lower confidence over time for old data
4. **Module Reputation Scoring** — Track which modules produce high-quality data
5. **Human Review Queue** — Automatic flagging of low-confidence content

---

**Next:** See [MIGRATIONS.md](./MIGRATIONS.md) for schema evolution and [RELATIONSHIPS.md](./RELATIONSHIPS.md) for relationship reference.
