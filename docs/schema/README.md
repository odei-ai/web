# ODEI Graph Schema Documentation

**Version:** 1.0
**Last Updated:** 2025-12-25
**Schema Version:** 1.3.0

---

## Overview

This directory contains comprehensive documentation for the ODEI Neo4j graph schema. The ODEI Memory Atlas is a **constitutional graph database** that organizes knowledge across 7 hierarchical layers with strict module-based access control.

**Key Statistics:**
- **38 node types** across 7 layers
- **67 relationship types** with layer constraints
- **4 operational modules** (discuss, plan, execute, mind)
- **Provenance tracking** on all writes

---

## Documentation Structure

### 1. [GRAPH-MODEL.md](./GRAPH-MODEL.md) — Visual Architecture

**Start here** for understanding the overall system.

**Contents:**
- 7-layer visual model with ASCII diagrams
- Layer-by-layer node type breakdown
- Complete node type reference with properties
- Relationship flow patterns
- Cross-layer connection rules

**Use when:**
- Learning the graph architecture
- Understanding which nodes go in which layers
- Finding node properties and constraints
- Planning new features

---

### 2. [QUERY-COOKBOOK.md](./QUERY-COOKBOOK.md) — Cypher Patterns

**Use this** for day-to-day development.

**Contents:**
- Common query patterns (find by layer, type, provenance)
- Layer-specific queries (foundation, vision, strategy, tactics, execution, track, mind)
- Multi-hop relationship traversal
- Performance optimization techniques
- Anti-patterns to avoid
- Complex query examples (workload assessment, OKR dashboards, etc.)

**Use when:**
- Writing Cypher queries
- Building dashboards
- Analyzing graph data
- Optimizing slow queries

---

### 3. [PROVENANCE.md](./PROVENANCE.md) — RBAC & Audit System

**Critical for understanding** who can modify what.

**Contents:**
- Provenance schema (module, actor, source, confidence)
- Module-based RBAC permission matrix
- Actor types and authority levels
- Confidence scoring guidelines
- Common patterns (human-created, agent-generated, joint)
- Audit and traceability queries

**Use when:**
- Creating nodes (provenance required on all writes)
- Understanding module permissions
- Debugging "forbidden operation" errors
- Auditing who created what

---

### 4. [RELATIONSHIPS.md](./RELATIONSHIPS.md) — Complete Reference

**Comprehensive reference** for all 67 relationship types.

**Contents:**
- Constitutional relationships (HOLDS_VALUE, ALIGNS_WITH, etc.)
- Directional relationships (SERVES, ADVANCES, CONTAINS)
- Learning relationships (INFORMS, VALIDATES, DEMONSTRATES)
- Time & capacity relationships (SCHEDULED_AS, LOGGED_AS)
- Relationship properties reference
- Layer crossing rules (what's allowed, what's forbidden)

**Use when:**
- Creating relationships
- Understanding semantic meaning of relationships
- Planning new relationship types
- Validating cross-layer connections

---

### 5. [MIGRATIONS.md](./MIGRATIONS.md) — Schema Evolution

**Read before changing the schema.**

**Contents:**
- How to add new node types
- How to add new relationships
- Safe changes vs breaking changes
- Backwards compatibility strategy
- Migration workflow (backup → test → apply → verify)
- Testing strategy
- Common migration patterns
- Rollback procedures

**Use when:**
- Adding new node types or relationships
- Modifying existing schema
- Planning breaking changes
- Writing migration scripts

---

## Quick Start

### For New Developers

1. **Read:** [GRAPH-MODEL.md](./GRAPH-MODEL.md) — Understand the 7-layer architecture
2. **Read:** [PROVENANCE.md](./PROVENANCE.md) — Learn RBAC and provenance requirements
3. **Bookmark:** [QUERY-COOKBOOK.md](./QUERY-COOKBOOK.md) — Use for daily queries
4. **Reference:** [RELATIONSHIPS.md](./RELATIONSHIPS.md) — Look up relationship types as needed

### For Schema Changes

1. **Plan:** Review [MIGRATIONS.md](./MIGRATIONS.md) — Understand safe vs breaking changes
2. **Check:** [GRAPH-MODEL.md](./GRAPH-MODEL.md) — Ensure new nodes fit the layer model
3. **Define:** Write Zod schemas in `/servers/odei-neo4j/src/domain/`
4. **Test:** Write migration scripts and unit tests
5. **Export:** Run `npm run schema:export` to update `.claude/SCHEMA.md`

### For Daily Usage

**Common tasks:**

```bash
# Export schema documentation
cd /Users/ai/ODEI/servers/odei-neo4j
npm run schema:export

# Backup database
npm run backup:neo4j

# Test schema changes
npm run build
npm test
```

**Common queries:**

```cypher
-- Find active goals
MATCH (g:Goal) WHERE g.status = 'active' RETURN g

-- Trace task to value
MATCH path = (t:Task)-[:SERVES*1..3]->(g:Goal)-[:ALIGNS_WITH]->(v:Value)
RETURN t.title, v.title

-- Workload assessment
MATCH (t:Task) WHERE t.status IN ['todo', 'in_progress']
RETURN sum(t.effortHours) AS total_effort
```

See [QUERY-COOKBOOK.md](./QUERY-COOKBOOK.md) for many more examples.

---

## Key Concepts

### 7-Layer Architecture

```
Layer 7: MIND       — Learning & Pattern Recognition
Layer 6: TRACK      — Observations & Metrics
Layer 5: EXECUTION  — Daily Actions & Tasks
Layer 4: TACTICS    — Projects & Systems
Layer 3: STRATEGY   — OKRs & Initiatives
Layer 2: VISION     — Direction & Goals
Layer 1: FOUNDATION — Constitutional Identity
```

**Design Principles:**
1. **Constitutional by Design** — Layer boundaries enforce separation of concerns
2. **Upward Traceability** — Everything traces back to Foundation
3. **Read-Only Observer** — Higher layers inform lower layers without modifying
4. **Module Ownership** — Each module owns specific layers

### Module-Based RBAC

| Module    | Write Access       | Purpose                     |
|-----------|-------------------|-----------------------------|
| `discuss` | foundation, vision | Constitutional dialogue     |
| `plan`    | strategy, tactics  | Strategic decomposition     |
| `execute` | tactics, execution, track | Task execution        |
| `mind`    | track, mind        | Pattern detection           |

See [PROVENANCE.md](./PROVENANCE.md) for complete permission matrix.

### Provenance (Required on All Writes)

```typescript
{
  module: "discuss" | "plan" | "execute" | "mind",
  actor: "human" | "agent" | "joint",
  source: string,      // Tool name, conversation ID
  confidence: number,  // 0.0-1.0
  notes?: string       // Optional context
}
```

**Authority hierarchy:**
1. `human` — Highest (direct human input)
2. `joint` — High (human-approved agent work)
3. `agent` — Medium (autonomous agent operation)

---

## Schema Versioning

**Current version:** 1.3.0 (auto-generated in `.claude/SCHEMA.md`)

**Version format:** `MAJOR.MINOR.PATCH`
- **MAJOR** — Breaking changes (requires migration)
- **MINOR** — New features (backwards-compatible)
- **PATCH** — Bug fixes (backwards-compatible)

**Update process:**
1. Modify schemas in `/servers/odei-neo4j/src/domain/`
2. Run `npm run build`
3. Run `npm run schema:export`
4. Commit changes with version bump

---

## Common Workflows

### Creating a New Node

```typescript
// Example: Create a Goal
await neo4j.goal.create.v1({
  data: {
    title: "Launch Tipz CBS MVP",
    summary: "Ship minimum viable product by Q1 2026",
    horizon: "quarter",
    deadline: "2026-03-31"
  },
  provenance: {
    module: "discuss",
    actor: "human",
    source: "conv_vision_planning_2025_12_25",
    confidence: 1.0
  }
});
```

### Creating a Relationship

```typescript
// Example: Connect Goal to Vision
await neo4j.relationship.create.v1({
  type: "SERVES",
  fromId: "goal_abc123",  // Goal ID
  toId: "vision_xyz456",   // Vision ID
  provenance: {
    module: "discuss",
    actor: "human",
    source: "manual_link",
    confidence: 1.0
  }
});
```

### Querying the Graph

```cypher
// Find all goals aligned with a value
MATCH (g:Goal)-[:ALIGNS_WITH]->(v:Value {title: 'Financial Freedom'})
WHERE g.status = 'active'
RETURN g.title, g.deadline
ORDER BY g.deadline
```

See [QUERY-COOKBOOK.md](./QUERY-COOKBOOK.md) for comprehensive examples.

---

## File References

### Auto-Generated Files

| File | Source | Update Method |
|------|--------|---------------|
| `.claude/SCHEMA.md` | `schemaInventory.ts` | `npm run schema:export` |
| `docs/schema/*.md` | Manual documentation | Edit directly |

### Source Code

| Directory | Purpose |
|-----------|---------|
| `servers/odei-neo4j/src/domain/` | Node types, relationships, constants |
| `servers/odei-neo4j/src/guardian/` | Validation, RBAC, errors |
| `servers/odei-neo4j/src/tools/` | MCP tool handlers |
| `servers/odei-neo4j/migrations/` | Schema migration scripts |

---

## Troubleshooting

### "Guardian Error: GUARD_FORBIDDEN_OPERATION"

**Cause:** Module trying to write to a layer it doesn't have permission for.

**Solution:** Check [PROVENANCE.md](./PROVENANCE.md) permission matrix. Use the correct module for the layer you're modifying.

```typescript
// ❌ WRONG: execute module trying to create foundation value
provenance: { module: "execute", ... }  // execute can't write foundation

// ✅ CORRECT: discuss module creating foundation value
provenance: { module: "discuss", ... }  // discuss owns foundation
```

### "Schema Error: summary is required"

**Cause:** All nodes require a `summary` field (enforced by APOC trigger).

**Solution:** Always provide a `summary` field (1-2000 chars):

```typescript
data: {
  title: "Goal Title",
  summary: "Brief description of the goal (1-2000 chars)",  // ✅ REQUIRED
  // ... other fields
}
```

### "Invalid relationship type"

**Cause:** Relationship type doesn't exist or isn't allowed between these node types.

**Solution:** Check [RELATIONSHIPS.md](./RELATIONSHIPS.md) for valid relationship types and layer pairings.

```cypher
-- ❌ WRONG: Task cannot modify Value
MATCH (t:Task), (v:Value)
CREATE (t)-[:MODIFIES]->(v)  // INVALID relationship

-- ✅ CORRECT: Task serves Goal
MATCH (t:Task), (g:Goal)
CREATE (t)-[:SERVES]->(g)  // VALID relationship
```

### "Metadata contains nested objects"

**Cause:** Neo4j only supports primitives in metadata fields.

**Solution:** Flatten or serialize complex objects:

```typescript
// ❌ WRONG
metadata: {
  config: { nested: { object: "value" } }  // Nested object
}

// ✅ CORRECT
metadata: {
  config: JSON.stringify({ nested: { object: "value" } })  // Serialized
}
```

---

## Best Practices

### ✅ DO

1. **Read schema docs** before making changes
2. **Use appropriate module** for the layer you're modifying
3. **Provide meaningful provenance** (source, notes)
4. **Test migrations** on backup data before production
5. **Follow layer boundaries** (don't cross forbidden boundaries)
6. **Include summaries** on all nodes (required field)
7. **Use LIMIT** in queries to prevent memory overflow

### ❌ DON'T

1. **Don't skip provenance** — Required for all writes
2. **Don't use nested metadata** — Only primitives allowed
3. **Don't cross forbidden layer boundaries** — Will be rejected
4. **Don't make schema changes without migrations** — Data corruption
5. **Don't use unbounded traversals** — Performance issues
6. **Don't bypass RBAC** — Use correct module for each layer

---

## Additional Resources

### Internal Documentation

- `/Users/ai/ODEI/.claude/SCHEMA.md` — Auto-generated schema (source of truth)
- `/Users/ai/ODEI/.claude/ARCHITECTURE.md` — Architecture overview
- `/Users/ai/ODEI/docs/architecture/7-LAYER-ARCHITECTURE.md` — Layer architecture spec
- `/Users/ai/ODEI/servers/odei-neo4j/docs/ARCHITECTURE.md` — MCP server architecture
- `/Users/ai/ODEI/servers/odei-neo4j/README.md` — Server README

### External Resources

- [Neo4j Cypher Manual](https://neo4j.com/docs/cypher-manual/current/)
- [Zod Documentation](https://zod.dev/)
- [JSON-RPC 2.0 Spec](https://www.jsonrpc.org/specification)

---

## Contributing

When adding new documentation:

1. **Update this README** — Add links to new files
2. **Follow existing structure** — Match formatting and style
3. **Include examples** — Show, don't just tell
4. **Cross-reference** — Link between related docs
5. **Regenerate schema** — Run `npm run schema:export` after changes

---

## Questions?

- Check existing docs first (especially [QUERY-COOKBOOK.md](./QUERY-COOKBOOK.md))
- Review auto-generated `.claude/SCHEMA.md` for latest schema
- Search codebase for examples: `grep -r "relationship.create" servers/odei-neo4j/src/`
- Ask in Discuss agent for constitutional questions
- Ask in Plan agent for strategic schema decisions

---

**Last Updated:** 2025-12-25
**Maintainer:** ODEI Infrastructure Team
**Status:** ✅ Active
