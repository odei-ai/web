# ODEI Graph Query Cookbook

**Version:** 1.0
**Generated:** 2025-12-25

---

## Table of Contents

1. [Common Query Patterns](#common-query-patterns)
2. [Layer-Specific Queries](#layer-specific-queries)
3. [Relationship Traversal](#relationship-traversal)
4. [Performance Optimization](#performance-optimization)
5. [Anti-Patterns to Avoid](#anti-patterns-to-avoid)
6. [Complex Query Examples](#complex-query-examples)

---

## Common Query Patterns

### Find All Active Nodes in a Layer

```cypher
// Get all active foundation nodes
MATCH (n)
WHERE n.layer = 'foundation' AND n.status = 'active'
RETURN n
```

### Find Nodes by Type

```cypher
// Get all goals
MATCH (n)
WHERE n.type = 'goal' AND n.status = 'active'
RETURN n
ORDER BY n.createdAt DESC
```

### Full-Text Search by Title

```cypher
// Search for nodes with "ODEI" in title (case-insensitive)
MATCH (n)
WHERE toLower(n.title) CONTAINS toLower('ODEI')
RETURN n.type, n.title, n.summary
ORDER BY n.layer, n.title
```

### Find Nodes by Provenance

```cypher
// Find all nodes created by a specific module
MATCH (n)
WHERE n.provenance.module = 'discuss'
RETURN n.layer, n.type, n.title
ORDER BY n.createdAt DESC
LIMIT 50
```

### Get Recent Nodes

```cypher
// Last 20 created nodes across all layers
MATCH (n)
WHERE n.status = 'active'
RETURN n.layer, n.type, n.title, n.createdAt
ORDER BY n.createdAt DESC
LIMIT 20
```

---

## Layer-Specific Queries

### Foundation Layer: Values & Principles

```cypher
// Get core values with supporting principles
MATCH (v:Value)-[r:SUPPORTED_BY]->(p:Principle)
WHERE v.priority = 'core'
RETURN v.title AS value,
       collect(p.title) AS principles
ORDER BY v.title
```

```cypher
// Get guardrails protecting specific principles
MATCH (pr:Principle)-[:ENFORCED_BY]->(g:Guardrail)
RETURN pr.title AS principle,
       g.title AS guardrail,
       g.trigger AS trigger,
       g.response AS response,
       g.severity AS severity
```

```cypher
// Find partnerships and what they enable
MATCH (p:Partnership)-[:ENABLES]->(target)
WHERE p.criticality = 'high'
RETURN p.title AS partnership,
       p.partner AS partner,
       labels(target) AS enabledType,
       target.title AS enabled
```

### Vision Layer: Goals & Business

```cypher
// Get active goals for current quarter
MATCH (g:Goal)
WHERE g.horizon = 'quarter'
  AND g.status = 'active'
  AND g.deadline >= date()
RETURN g.title, g.target, g.deadline
ORDER BY g.deadline
```

```cypher
// Get business metrics
MATCH (b:Business)-[:HAS_METRIC]->(m:Metric)
RETURN b.title AS business,
       collect({
         metric: m.title,
         unit: m.unit,
         direction: m.direction
       }) AS metrics
```

```cypher
// Trace goal alignment to values
MATCH path = (g:Goal)-[:ALIGNS_WITH]->(v:Value)
WHERE g.status = 'active'
RETURN g.title AS goal,
       v.title AS value,
       length(path) AS alignment_depth
```

### Strategy Layer: OKRs

```cypher
// Get objectives with key results
MATCH (o:Objective)-[:HAS_KEY_RESULT]->(kr:KeyResult)
WHERE o.status = 'active'
RETURN o.title AS objective,
       collect({
         keyResult: kr.title,
         metric: kr.metric,
         target: kr.target
       }) AS keyResults
```

```cypher
// Get initiative breakdown with milestones
MATCH (i:Initiative)-[:HAS_MILESTONE]->(m:Milestone)
WHERE i.status = 'active'
OPTIONAL MATCH (m)-[:DEPENDS_ON]->(prev:Milestone)
RETURN i.title AS initiative,
       m.title AS milestone,
       m.deadline AS deadline,
       m.status AS status,
       collect(prev.title) AS dependencies
ORDER BY m.deadline
```

```cypher
// Risk assessment for active initiatives
MATCH (i:Initiative)-[:HAS_RISK]->(r:Risk)
WHERE i.status = 'active'
RETURN i.title AS initiative,
       collect({
         risk: r.title,
         probability: r.probability,
         impact: r.impact,
         mitigation: r.mitigation
       }) AS risks
ORDER BY r.probability * (CASE r.impact
  WHEN 'high' THEN 1.0
  WHEN 'medium' THEN 0.5
  ELSE 0.2 END) DESC
```

### Tactics Layer: Projects & Tasks

```cypher
// Get project task breakdown
MATCH (p:Project)-[:HAS_TASK]->(t:Task)
WHERE p.status = 'in_progress'
RETURN p.title AS project,
       count(t) AS total_tasks,
       count(CASE WHEN t.status = 'done' THEN 1 END) AS completed,
       count(CASE WHEN t.status = 'in_progress' THEN 1 END) AS in_progress,
       count(CASE WHEN t.status = 'blocked' THEN 1 END) AS blocked
```

```cypher
// Find project dependencies (critical path)
MATCH (p1:Project)-[:DEPENDS_ON]->(p2:Project)
WHERE p1.status IN ['active', 'in_progress']
RETURN p1.title AS project,
       p2.title AS depends_on,
       p2.status AS blocker_status
ORDER BY p1.title
```

```cypher
// Get recurring processes and generated tasks
MATCH (pr:Process)-[:GENERATES_TASK]->(t:Task)
WHERE pr.status = 'active'
RETURN pr.title AS process,
       pr.cadence AS cadence,
       count(t) AS tasks_generated,
       max(t.createdAt) AS last_generated
```

### Execution Layer: Tasks & Time

```cypher
// Get upcoming scheduled work
MATCH (t:Task)-[:SCHEDULED_AS]->(tb:TimeBlock)
WHERE tb.start >= datetime()
  AND t.status IN ['todo', 'in_progress']
RETURN t.title AS task,
       tb.start AS scheduled_start,
       tb.end AS scheduled_end,
       t.effortHours AS estimated_hours
ORDER BY tb.start
LIMIT 10
```

```cypher
// Work session analytics
MATCH (t:Task)-[:LOGGED_AS]->(ws:WorkSession)
WHERE ws.start >= datetime() - duration('P7D')
RETURN t.title AS task,
       count(ws) AS sessions,
       sum(duration.between(datetime(ws.start), datetime(ws.end)).hours) AS total_hours
ORDER BY total_hours DESC
```

```cypher
// Decision impact analysis
MATCH (d:Decision)-[:ADVANCES]->(target)
WHERE d.decidedAt >= datetime() - duration('P30D')
RETURN d.title AS decision,
       d.roi.total AS roi_score,
       labels(target) AS impacts,
       collect(target.title) AS affected
ORDER BY d.roi.total DESC
```

### Track Layer: Metrics & Observations

```cypher
// Metric tracking over time
MATCH (obs:Observation)-[:TRACKS]->(m:Metric)
WHERE m.title = 'Daily Active Users'
  AND obs.observedAt >= datetime() - duration('P30D')
RETURN obs.observedAt AS date,
       obs.summary AS value
ORDER BY obs.observedAt DESC
```

```cypher
// Goal observation dashboard
MATCH (g:Goal)<-[:OBSERVES]-(obs:Observation)
WHERE g.status = 'active'
RETURN g.title AS goal,
       count(obs) AS observation_count,
       max(obs.observedAt) AS last_observed
ORDER BY last_observed DESC
```

### Mind Layer: Insights & Patterns

```cypher
// High-confidence insights
MATCH (ins:Insight)
WHERE ins.confidence >= 0.8
  AND ins.status = 'active'
OPTIONAL MATCH (ins)-[:APPLIED_TO]->(target)
RETURN ins.title AS insight,
       ins.confidence AS confidence,
       collect(target.title) AS applied_to
ORDER BY ins.confidence DESC
```

```cypher
// Pattern validation chain
MATCH (e:Evidence)-[:VALIDATES]->(p:Pattern)-[:DEMONSTRATES]->(v:Value)
RETURN v.title AS value,
       p.title AS pattern,
       p.confidence AS confidence,
       collect(e.title) AS evidence
ORDER BY p.confidence DESC
```

```cypher
// Insight → Decision traceability
MATCH (ins:Insight)-[:TRIGGERS]->(d:Decision)
WHERE d.decidedAt >= datetime() - duration('P90D')
RETURN ins.title AS insight,
       d.title AS decision,
       d.decidedAt AS when_decided,
       d.roi.total AS roi_score
ORDER BY d.decidedAt DESC
```

---

## Relationship Traversal

### Multi-Hop Traceability

```cypher
// Task → Goal → Value (full upward trace)
MATCH path = (t:Task)-[:SERVES*1..3]->(g:Goal)-[:ALIGNS_WITH]->(v:Value)
WHERE t.status = 'in_progress'
RETURN t.title AS task,
       [node IN nodes(path) | node.title] AS trace_path,
       v.title AS constitutional_value
```

### Variable-Length Relationships

```cypher
// Find all descendants of a goal (any depth)
MATCH (g:Goal {id: 'goal_500m_10y'})-[*1..]->(descendant)
RETURN DISTINCT labels(descendant) AS type,
       descendant.title AS title,
       descendant.layer AS layer
ORDER BY descendant.layer, type
```

### Bidirectional Search

```cypher
// Find connections between two nodes (any direction)
MATCH path = shortestPath(
  (n1 {id: 'value_autonomy'})-[*1..5]-(n2 {id: 'task_build_ui'})
)
RETURN [node IN nodes(path) | node.title] AS connection_path,
       [rel IN relationships(path) | type(rel)] AS relationships
```

---

## Performance Optimization

### Use Indexes Effectively

```cypher
// Index on layer + status (fast filtering)
CREATE INDEX node_layer_status IF NOT EXISTS
FOR (n) REQUIRE (n.layer, n.status)
```

```cypher
// Index on type (common filter)
CREATE INDEX node_type IF NOT EXISTS
FOR (n) REQUIRE n.type
```

### Limit Result Sets

```cypher
// Always use LIMIT for large result sets
MATCH (n)
WHERE n.layer = 'execution'
RETURN n
ORDER BY n.createdAt DESC
LIMIT 100  // ✅ GOOD: Prevents memory overflow
```

### Profile Slow Queries

```cypher
// Use PROFILE to see execution plan
PROFILE
MATCH (g:Goal)-[:HAS_PROJECT*1..3]->(p:Project)
WHERE g.status = 'active'
RETURN g.title, count(p) AS project_count
```

### Batch Updates

```cypher
// Efficient batch update with UNWIND
UNWIND $batch AS item
MATCH (n {id: item.id})
SET n.status = item.status,
    n.updatedAt = datetime()
RETURN count(n) AS updated
```

---

## Anti-Patterns to Avoid

### ❌ Anti-Pattern 1: Unbounded Traversal

```cypher
// BAD: No depth limit
MATCH (n)-[*]->(m)
RETURN count(m)  // May traverse entire graph!
```

```cypher
// GOOD: Bounded depth
MATCH (n)-[*1..5]->(m)
WHERE n.id = 'specific_node'
RETURN count(m)
```

### ❌ Anti-Pattern 2: Missing WHERE Filters

```cypher
// BAD: Scans all nodes
MATCH (n)
RETURN n.title
```

```cypher
// GOOD: Filter early
MATCH (n)
WHERE n.layer = 'foundation' AND n.status = 'active'
RETURN n.title
```

### ❌ Anti-Pattern 3: Cartesian Products

```cypher
// BAD: Creates cartesian product
MATCH (g:Goal), (p:Project)
RETURN g, p  // Returns every goal × every project!
```

```cypher
// GOOD: Use relationship
MATCH (g:Goal)-[:HAS_PROJECT]->(p:Project)
RETURN g, p
```

### ❌ Anti-Pattern 4: Expensive COLLECT Without LIMIT

```cypher
// BAD: May collect millions of items
MATCH (n)-[:HAS_TASK]->(t:Task)
RETURN n.title, collect(t.title) AS all_tasks
```

```cypher
// GOOD: Limit collection size
MATCH (n)-[:HAS_TASK]->(t:Task)
WITH n, t
ORDER BY t.createdAt DESC
LIMIT 20
RETURN n.title, collect(t.title) AS recent_tasks
```

### ❌ Anti-Pattern 5: String Pattern Matching Without Index

```cypher
// BAD: Full table scan
MATCH (n)
WHERE n.title =~ '.*ODEI.*'
RETURN n
```

```cypher
// GOOD: Use CONTAINS (can use index)
MATCH (n)
WHERE n.title CONTAINS 'ODEI'
RETURN n
```

---

## Complex Query Examples

### Workload Assessment

```cypher
// Calculate total effort hours for active tasks
MATCH (t:Task)
WHERE t.status IN ['todo', 'in_progress']
  AND t.effortHours IS NOT NULL
OPTIONAL MATCH (t)-[:SCHEDULED_AS]->(tb:TimeBlock)
RETURN sum(t.effortHours) AS total_effort_hours,
       sum(CASE WHEN tb IS NOT NULL THEN t.effortHours ELSE 0 END) AS scheduled_hours,
       sum(CASE WHEN tb IS NULL THEN t.effortHours ELSE 0 END) AS unscheduled_hours,
       count(t) AS total_tasks
```

### Goal Progress Dashboard

```cypher
// Multi-layer goal progress
MATCH (g:Goal)
WHERE g.status = 'active' AND g.horizon = 'quarter'
OPTIONAL MATCH (g)<-[:SERVES]-(o:Objective)
OPTIONAL MATCH (o)-[:HAS_KEY_RESULT]->(kr:KeyResult)
OPTIONAL MATCH (g)-[:HAS_PROJECT]->(p:Project)
OPTIONAL MATCH (p)-[:HAS_TASK]->(t:Task)
RETURN g.title AS goal,
       g.deadline AS deadline,
       count(DISTINCT o) AS objectives,
       count(DISTINCT kr) AS key_results,
       count(DISTINCT p) AS projects,
       count(DISTINCT t) AS tasks,
       count(CASE WHEN t.status = 'done' THEN 1 END) AS tasks_complete,
       CASE
         WHEN count(t) > 0
         THEN round(100.0 * count(CASE WHEN t.status = 'done' THEN 1 END) / count(t), 1)
         ELSE 0
       END AS completion_percent
ORDER BY g.deadline
```

### Constitutional Alignment Report

```cypher
// Verify all active goals align with values
MATCH (g:Goal)
WHERE g.status = 'active'
OPTIONAL MATCH (g)-[:ALIGNS_WITH]->(v:Value)
WITH g, collect(v.title) AS aligned_values
RETURN g.title AS goal,
       g.horizon AS horizon,
       CASE
         WHEN size(aligned_values) = 0 THEN 'UNALIGNED'
         ELSE 'ALIGNED'
       END AS alignment_status,
       aligned_values
ORDER BY alignment_status, g.horizon
```

### Learning Feedback Loop

```cypher
// Observations → Insights → Applied Changes
MATCH (obs:Observation)-[:INFORMS]->(ins:Insight)
WHERE obs.observedAt >= datetime() - duration('P30D')
OPTIONAL MATCH (ins)-[:APPLIED_TO]->(target)
OPTIONAL MATCH (ins)-[:TRIGGERS]->(d:Decision)
RETURN ins.title AS insight,
       ins.confidence AS confidence,
       count(DISTINCT obs) AS observations,
       collect(DISTINCT target.title) AS applied_to,
       collect(DISTINCT d.title) AS triggered_decisions
ORDER BY ins.confidence DESC, count(obs) DESC
```

### Capacity Planning

```cypher
// Weekly capacity vs committed hours
MATCH (tb:TimeBlock)
WHERE tb.start >= datetime()
  AND tb.start < datetime() + duration('P7D')
OPTIONAL MATCH (t:Task)-[:SCHEDULED_AS]->(tb)
WITH tb, t
RETURN date(tb.start) AS day,
       sum(duration.between(datetime(tb.start), datetime(tb.end)).hours) AS available_hours,
       sum(CASE WHEN t IS NOT NULL THEN t.effortHours ELSE 0 END) AS committed_hours,
       count(DISTINCT t) AS scheduled_tasks
ORDER BY day
```

### Risk-Weighted Initiative Prioritization

```cypher
// Rank initiatives by impact vs risk
MATCH (i:Initiative)
WHERE i.status = 'active'
OPTIONAL MATCH (i)-[:HAS_RISK]->(r:Risk)
WITH i,
     avg(r.probability * CASE r.impact
       WHEN 'high' THEN 1.0
       WHEN 'medium' THEN 0.5
       ELSE 0.2 END) AS avg_risk_score,
     CASE i.impact
       WHEN 'high' THEN 3
       WHEN 'medium' THEN 2
       ELSE 1
     END AS impact_score
RETURN i.title AS initiative,
       i.impact AS impact,
       avg_risk_score AS risk_score,
       impact_score - coalesce(avg_risk_score, 0) AS priority_score
ORDER BY priority_score DESC
```

### Evidence-Based Pattern Discovery

```cypher
// Find patterns with strongest evidence
MATCH (e:Evidence)-[:VALIDATES]->(p:Pattern)
WITH p, count(e) AS evidence_count, collect(e.title) AS evidence_list
WHERE evidence_count >= 3
OPTIONAL MATCH (p)-[:DEMONSTRATES]->(v:Value)
RETURN p.title AS pattern,
       p.confidence AS confidence,
       evidence_count AS evidence_count,
       evidence_list[0..5] AS sample_evidence,
       collect(v.title) AS demonstrates_values
ORDER BY evidence_count DESC, p.confidence DESC
```

---

## Query Templates by Use Case

### Daily Standup Query

```cypher
// Today's focus
MATCH (t:Task)
WHERE t.status IN ['in_progress', 'todo']
  AND (t.dueDate IS NULL OR date(t.dueDate) <= date())
OPTIONAL MATCH (t)-[:SCHEDULED_AS]->(tb:TimeBlock)
WHERE tb.start >= datetime()
  AND tb.start < datetime() + duration('P1D')
RETURN t.title AS task,
       t.effortHours AS effort,
       tb.start AS scheduled,
       t.priority AS priority
ORDER BY COALESCE(t.priority, 'p3'), tb.start
```

### Weekly Review Query

```cypher
// Last 7 days accomplishments
MATCH (t:Task)
WHERE t.status = 'done'
  AND t.updatedAt >= datetime() - duration('P7D')
OPTIONAL MATCH (t)-[:SERVES]->(g:Goal)
OPTIONAL MATCH (t)<-[:HAS_TASK]-(p:Project)
RETURN date(t.updatedAt) AS completed_date,
       t.title AS task,
       p.title AS project,
       g.title AS goal,
       t.effortHours AS hours
ORDER BY t.updatedAt DESC
```

### Monthly OKR Review

```cypher
// Current quarter OKR status
MATCH (o:Objective)-[:HAS_KEY_RESULT]->(kr:KeyResult)
WHERE o.status = 'active'
  AND o.timeframe CONTAINS toString(year(date()))
OPTIONAL MATCH (kr)<-[:ADVANCES]-(d:Decision)
WHERE d.decidedAt >= datetime() - duration('P30D')
RETURN o.title AS objective,
       kr.title AS key_result,
       kr.metric AS metric,
       kr.target AS target,
       count(d) AS decisions_this_month
ORDER BY o.title, kr.title
```

---

**Next:** See [PROVENANCE.md](./PROVENANCE.md) for provenance tracking and [MIGRATIONS.md](./MIGRATIONS.md) for schema evolution patterns.
