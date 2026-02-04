# ODEI 7-Layer Architecture

**Version:** 1.0
**Date:** November 5, 2025
**Status:** Production

---

## Executive Summary

ODEI implements a **7-layer knowledge graph architecture** that separates concerns across identity, direction, strategy, tactics, execution, tracking, and learning. Each layer contains specific node types and is managed by dedicated agents to prevent conflicts, enable parallel development, and scale to $500M operations.

**Key Principle:** _Agents own their layers. Cross-layer access is read-only via search._

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│  Layer 7: MIND (Insights & Patterns)                        │
│  Agent: Mind | Nodes: Insight, Pattern, Evidence            │
└──────────────────────┬──────────────────────────────────────┘
                       ↑ INFORMS
┌──────────────────────┴──────────────────────────────────────┐
│  Layer 6: TRACK (Metrics & Observations)                    │
│  Agent: Mind | Nodes: Metric, Observation, Event, Signal    │
└──────────────────────┬──────────────────────────────────────┘
                       ↑ OBSERVES
┌──────────────────────┴──────────────────────────────────────┐
│  Layer 5: EXECUTION (Actions & Tasks)                       │
│  Agent: Execute | Nodes: Decision, Task, TimeBlock,         │
│                         WorkSession, Action                  │
└──────────────────────┬──────────────────────────────────────┘
                       ↑ HAS_TASK, SERVES
┌──────────────────────┴──────────────────────────────────────┐
│  Layer 4: TACTICS (Projects & Systems)                      │
│  Agent: Execute | Nodes: Project, Area, System, Process     │
└──────────────────────┬──────────────────────────────────────┘
                       ↑ HAS_PROJECT
┌──────────────────────┴──────────────────────────────────────┐
│  Layer 3: STRATEGY (Objectives & Initiatives)               │
│  Agent: Decisions | Nodes: Objective, KeyResult,            │
│                            Initiative, Risk                  │
└──────────────────────┬──────────────────────────────────────┘
                       ↑ SERVES, ADVANCES
┌──────────────────────┴──────────────────────────────────────┐
│  Layer 2: VISION (Direction & Goals)                        │
│  Agent: Discuss | Nodes: Vision, Business, Goal, Season     │
└──────────────────────┬──────────────────────────────────────┘
                       ↑ ALIGNS_WITH, SERVES
┌──────────────────────┴──────────────────────────────────────┐
│  Layer 1: FOUNDATION (Identity & Values)                    │
│  Agent: Discuss | Nodes: Value, Principle, Guardrail,       │
│                         Human, AI, Context, Policy           │
└─────────────────────────────────────────────────────────────┘
```

---

## Layer Definitions

### Layer 1: Foundation (Identity)

**Purpose:** WHO WE ARE — constitutional identity and values

**Node Types:**

- `Value` — Core beliefs and principles we hold (e.g., "Autonomy", "Ship fast")
- `Principle` — Operating rules derived from values (e.g., "Make reversible decisions quickly")
- `Guardrail` — Non-negotiable boundaries with triggers (e.g., "No evening strategic pivots")
- `Human` — Human participants (Anton's profile, background, constraints)
- `AI` — AI agent identities and capabilities
- `Context` — Contextual framing (timezone, environment, constraints)
- `Policy` — Established policies and their scope

**Managed By:** Discuss agent

**Relationships:**

- `HOLDS_VALUE` (Human → Value)
- `FOLLOWS_PRINCIPLE` (Human → Principle)
- `RESPECTS_GUARDRAIL` (Human → Guardrail)
- `SUPPORTED_BY` (Value → Principle)
- `ENFORCED_BY` (Principle → Guardrail)
- `ALIGNS_WITH` (Value/Principle → Vision/Goal in Layer 2)

**Examples:**

```cypher
// Value node
CREATE (v:Value {
  id: 'val_autonomy',
  title: 'Autonomy',
  statement: 'We operate independently without external control',
  type: 'core',
  layer: 'foundation'
})

// Principle node supporting Value
CREATE (p:Principle {
  id: 'pr_quick_decisions',
  title: 'Make reversible decisions quickly',
  statement: 'For low-risk decisions, decide in <5 minutes',
  layer: 'foundation'
})
CREATE (v)-[:SUPPORTED_BY]->(p)
```

---

### Layer 2: Vision (Direction)

**Purpose:** WHERE WE GO — long-term direction and goals

**Node Types:**

- `Vision` — Directional statements across time horizons (Life → 10y → Annual → Quarter → Month)
- `Goal` — Hierarchical goals (Decade → Year → Quarter → Month → Week → Day)
- `Business` — Operating companies that operationalize the life vision (e.g., Tipz CBS, Automated Exchange)
- `Season` — Temporal execution windows (e.g., "Q4 2025: Ship ODEI v1")

**Managed By:** Discuss agent

**Relationships:**

- `HAS_GOAL` (Season → Goal)
- `PURSUES_GOAL` (Human → Goal)
- `SERVES` (Goal → Vision)
- `SUPPORTED_BY` (Goal → parent Goal)
- `ALIGNS_WITH` (Goal → Value/Principle in Layer 1)

**Examples:**

```cypher
// Vision node
CREATE (v:Vision {
  id: 'vis_10y',
  title: '10-Year Vision: $100M AI-native company',
  horizon: '10y',
  layer: 'vision'
})

// Goal serving Vision
CREATE (g:Goal {
  id: 'goal_q4_2025',
  title: 'Ship ODEI MVP to 100 users',
  timeframe: 'quarter',
  layer: 'vision'
})
CREATE (g)-[:SERVES]->(v)
```

---

### Layer 3: Strategy (How We Get There)

**Purpose:** HOW WE ACHIEVE IT — strategic planning and ROI analysis

**Node Types:**

- `Objective` — Strategic objectives with measurable outcomes
- `KeyResult` — Quantified results for objectives (OKR framework)
- `Initiative` — Strategic initiatives advancing objectives
- `Risk` — Identified risks with mitigation plans

**Managed By:** Decisions agent

**Relationships:**

- `SERVES` (Objective → Goal in Layer 2)
- `HAS_KEY_RESULT` (Objective → KeyResult)
- `ADVANCES` (Initiative → Objective)
- `SERVES` (Initiative → Goal in Layer 2)
- `HAS_PROJECT` (Initiative → Project in Layer 4)

**Examples:**

```cypher
// Objective serving Goal
CREATE (o:Objective {
  id: 'obj_odei_launch',
  title: 'Launch ODEI to market',
  layer: 'strategy'
})
CREATE (o)-[:SERVES]->(g:Goal {id: 'goal_q4_2025'})

// KeyResult under Objective
CREATE (kr:KeyResult {
  id: 'kr_100_users',
  title: '100 active users by Dec 31',
  metric: 'user_count',
  target: 100,
  layer: 'strategy'
})
CREATE (o)-[:HAS_KEY_RESULT]->(kr)
```

---

### Layer 4: Tactics (Projects & Systems)

**Purpose:** WHAT WE BUILD — project management and systems

**Node Types:**

- `Project` — Discrete projects with deliverables
- `Area` — Ongoing areas of responsibility
- `System` — Established systems and processes
- `Process` — Documented processes

**Managed By:** Execute agent

**Relationships:**

- `HAS_PROJECT` (Goal in Layer 2 → Project OR Initiative in Layer 3 → Project)
- `HAS_TASK` (Project → Task in Layer 5)
- `DEPENDS_ON` (Project → Project)
- `SUPPORTS` (Project → Project)

**Examples:**

```cypher
// Project under Initiative
CREATE (p:Project {
  id: 'proj_electron_ui',
  title: 'Build Electron UI for ODEI',
  status: 'in_progress',
  layer: 'tactics'
})
CREATE (i:Initiative {id: 'init_odei_mvp'})-[:HAS_PROJECT]->(p)
```

---

### Layer 5: Execution (Actions & Tasks)

**Purpose:** WHAT WE DO — daily execution and time management

**Node Types:**

- `Decision` — Recorded decisions with rationale
- `Task` — Actionable tasks with effort estimates
- `TimeBlock` — Scheduled time blocks on calendar
- `WorkSession` — Logged work sessions
- `Action` — Atomic actions (smallest unit of work)

**Managed By:** Execute agent

**Relationships:**

- `SCHEDULED_AS` (Task → TimeBlock)
- `LOGGED_AS` (Task → WorkSession)
- `SERVES` (Decision/Task → Goal in Layer 2)
- `SERVES` (Decision → Project in Layer 4)
- `ADVANCES` (Decision → Objective/KeyResult/Initiative in Layer 3)

**Examples:**

```cypher
// Task under Project
CREATE (t:Task {
  id: 'task_build_agent_ui',
  title: 'Build agent panel UI component',
  status: 'todo',
  effortHours: 8,
  layer: 'execution'
})
CREATE (p:Project {id: 'proj_electron_ui'})-[:HAS_TASK]->(t)

// TimeBlock scheduling Task
CREATE (tb:TimeBlock {
  id: 'tb_monday_morning',
  start: '2025-11-06T09:00:00Z',
  end: '2025-11-06T13:00:00Z',
  layer: 'execution'
})
CREATE (t)-[:SCHEDULED_AS]->(tb)
```

---

### Layer 6: Track (Metrics & Observations)

**Purpose:** WHAT WE MEASURE — performance tracking and observations

**Node Types:**

- `Metric` — Quantitative metrics (e.g., "Daily active users")
- `Observation` — Qualitative observations
- `Event` — Significant events (launches, milestones)
- `Signal` — Early signals and indicators

**Managed By:** Mind agent

**Relationships:**

- `TRACKS` (Observation → Metric)
- `OBSERVES` (Observation → Goal in Layer 2 OR Project in Layer 4)
- `INFORMS` (Observation → Insight/Pattern in Layer 7)

**Examples:**

```cypher
// Metric tracking Goal
CREATE (m:Metric {
  id: 'metric_dau',
  title: 'Daily Active Users',
  type: 'quantitative',
  unit: 'users',
  layer: 'track'
})

// Observation tracking Metric
CREATE (obs:Observation {
  id: 'obs_nov_5_dau',
  value: 42,
  timestamp: '2025-11-05T12:00:00Z',
  layer: 'track'
})
CREATE (obs)-[:TRACKS]->(m)
```

---

### Layer 7: Mind (Insights & Patterns)

**Purpose:** WHAT WE LEARN — learning and pattern recognition

**Node Types:**

- `Insight` — Discovered insights from data
- `Pattern` — Recurring patterns observed
- `Evidence` — Supporting evidence for insights

**Managed By:** Mind agent

**Relationships:**

- `APPLIED_TO` (Insight → Goal in Layer 2 OR Project in Layer 4)
- `TRIGGERS` (Insight → Decision in Layer 5)
- `SUPPORTED_BY` (Decision in Layer 5 → Evidence)
- `INFORMS` (Observation in Layer 6 → Insight/Pattern)

**Examples:**

```cypher
// Pattern discovered from Observations
CREATE (pat:Pattern {
  id: 'pat_evening_critic',
  title: 'Evening strategic pivots fail 87% of time',
  confidence: 0.87,
  sampleSize: 23,
  layer: 'mind'
})

// Insight derived from Pattern
CREATE (ins:Insight {
  id: 'ins_morning_strategy',
  title: 'Strategic decisions should happen 09:00-14:00',
  layer: 'mind'
})
CREATE (obs:Observation)-[:INFORMS]->(ins)
```

---

## Agent Ownership & Responsibilities

### Discuss Agent (Layers 1-2)

**Manages:**

- Foundation layer (Layer 1): Values, Principles, Guardrails, Human/AI identity
- Vision layer (Layer 2): Vision, Business, Goals, Seasons

**Can Read:**

- Layers 1-2: Full access
- Layers 3-7: Summary data only via `workload.assess.v1` (capacity calculation)

**Can Write:**

- Layers 1-2: Create, update, archive nodes
- Layers 3-7: ❌ NO WRITE ACCESS

**Why This Scope:**

- Discuss focuses on WHO WE ARE and WHERE WE GO
- Constitutional decisions require deep context on identity and direction
- Tasks/Projects/Strategy are downstream execution (other agents)

**Example Valid Operations:**

```cypher
// ✅ CREATE new Value (Layer 1)
CREATE (v:Value {title: 'Autonomy', layer: 'foundation'})

// ✅ CREATE new Goal serving Vision (Layer 2)
CREATE (g:Goal {title: 'Q4: Ship ODEI', layer: 'vision'})
CREATE (g)-[:SERVES]->(v:Vision {id: 'vis_annual'})

// ✅ READ workload summary (across all layers)
CALL workload.assess.v1({includeStatuses: ['todo', 'in_progress']})

// ❌ CANNOT create Task (Layer 5 - Execute territory)
CREATE (t:Task {title: 'Build feature X', layer: 'execution'})  // BLOCKED
```

---

### Decisions Agent (Layer 3)

**Manages:**

- Strategy layer (Layer 3): Objectives, KeyResults, Initiatives, Risks

**Can Read:**

- Layers 1-2: Full access (needs context for strategic alignment)
- Layer 3: Full access (owns this layer)
- Layers 4-7: Search access for impact analysis

**Can Write:**

- Layer 3: Create, update, archive nodes
- Layers 1-2, 4-7: ❌ NO WRITE ACCESS

**Why This Scope:**

- Decisions focuses on HOW WE ACHIEVE goals
- Requires Foundation/Vision context to validate alignment
- Cannot modify identity (Layer 1) or execution details (Layers 4-5)

**Example Valid Operations:**

```cypher
// ✅ CREATE Objective serving Goal (Layer 3)
CREATE (o:Objective {title: 'Achieve PMF', layer: 'strategy'})
CREATE (o)-[:SERVES]->(g:Goal {id: 'goal_q4'})

// ✅ READ Foundation for alignment validation
MATCH (v:Value)-[:ALIGNS_WITH]->(g:Goal {id: 'goal_q4'})
RETURN v

// ❌ CANNOT modify Goal (Layer 2 - Discuss territory)
MATCH (g:Goal {id: 'goal_q4'})
SET g.title = 'New title'  // BLOCKED
```

---

### Execute Agent (Layers 4-5)

**Manages:**

- Tactics layer (Layer 4): Projects, Areas, Systems, Processes
- Execution layer (Layer 5): Decisions, Tasks, TimeBlocks, WorkSessions

**Can Read:**

- Layers 1-3: Search access (needs context for task prioritization)
- Layers 4-5: Full access (owns these layers)
- Layers 6-7: Summary metrics (performance tracking)

**Can Write:**

- Layers 4-5: Create, update, archive nodes
- Layers 1-3, 6-7: ❌ NO WRITE ACCESS

**Why This Scope:**

- Execute focuses on WHAT WE DO daily
- Requires Strategy/Goals context to prioritize tasks
- Cannot modify constitution (Layer 1) or strategy (Layer 3)

**Example Valid Operations:**

```cypher
// ✅ CREATE Project under Initiative (Layer 4)
CREATE (p:Project {title: 'Build UI', layer: 'tactics'})
CREATE (i:Initiative {id: 'init_mvp'})-[:HAS_PROJECT]->(p)

// ✅ CREATE Task under Project (Layer 5)
CREATE (t:Task {title: 'Design agent panel', effortHours: 6, layer: 'execution'})
CREATE (p)-[:HAS_TASK]->(t)

// ❌ CANNOT modify Objective (Layer 3 - Decisions territory)
MATCH (o:Objective {id: 'obj_pmf'})
SET o.status = 'completed'  // BLOCKED
```

---

### Mind Agent (Layers 6-7)

**Manages:**

- Track layer (Layer 6): Metrics, Observations, Events, Signals
- Mind layer (Layer 7): Insights, Patterns, Evidence

**Can Read:**

- Layers 1-7: Full access (needs complete context for pattern analysis)

**Can Write:**

- Layers 6-7: Create, update, archive nodes
- Layers 1-5: ❌ NO WRITE ACCESS

**Why This Scope:**

- Mind focuses on WHAT WE LEARN from all layers
- Requires read access to everything for pattern detection
- Cannot modify goals/tasks/strategy (read-only observer)

**Example Valid Operations:**

```cypher
// ✅ CREATE Observation tracking Goal (Layer 6)
CREATE (obs:Observation {
  value: 'Goal completion rate dropped to 65%',
  timestamp: '2025-11-05',
  layer: 'track'
})
CREATE (obs)-[:OBSERVES]->(g:Goal {id: 'goal_week_42'})

// ✅ CREATE Pattern from Observations (Layer 7)
CREATE (pat:Pattern {
  title: 'Week goals fail when capacity >85%',
  confidence: 0.82,
  layer: 'mind'
})
CREATE (obs)-[:INFORMS]->(pat)

// ❌ CANNOT modify Task (Layer 5 - Execute territory)
MATCH (t:Task {id: 'task_123'})
SET t.status = 'completed'  // BLOCKED
```

---

## Cross-Layer Relationship Patterns

### Foundation → Vision (Layer 1 → 2)

**Relationships:**

- `ALIGNS_WITH` (Value/Principle → Vision/Goal)

**Purpose:** Ensure goals align with constitutional values

**Example:**

```cypher
// Value aligns with Vision
MATCH (v:Value {title: 'Autonomy'})
MATCH (vis:Vision {title: '10-Year: Independent $100M company'})
CREATE (v)-[:ALIGNS_WITH]->(vis)

// Goal aligns with Principle
MATCH (p:Principle {title: 'Ship fast, iterate'})
MATCH (g:Goal {title: 'MVP in 90 days'})
CREATE (g)-[:ALIGNS_WITH]->(p)
```

---

### Vision → Strategy (Layer 2 → 3)

**Relationships:**

- `SERVES` (Objective → Goal)
- `SERVES` (Initiative → Goal)

**Purpose:** Strategic objectives serve directional goals

**Example:**

```cypher
// Objective serves Goal
MATCH (g:Goal {title: 'Q4: Ship ODEI MVP'})
CREATE (o:Objective {title: 'Achieve product-market fit', layer: 'strategy'})
CREATE (o)-[:SERVES]->(g)

// Initiative serves Goal
CREATE (init:Initiative {title: 'Build Electron UI', layer: 'strategy'})
CREATE (init)-[:SERVES]->(g)
```

---

### Strategy → Tactics (Layer 3 → 4)

**Relationships:**

- `HAS_PROJECT` (Initiative → Project)

**Purpose:** Initiatives decompose into projects

**Example:**

```cypher
// Initiative has Project
MATCH (init:Initiative {title: 'Build Electron UI'})
CREATE (p:Project {title: 'Agent panel component', layer: 'tactics'})
CREATE (init)-[:HAS_PROJECT]->(p)
```

---

### Tactics → Execution (Layer 4 → 5)

**Relationships:**

- `HAS_TASK` (Project → Task)
- `SERVES` (Task → Project)

**Purpose:** Projects break down into tasks

**Example:**

```cypher
// Project has Tasks
MATCH (p:Project {title: 'Agent panel component'})
CREATE (t1:Task {title: 'Design mockups', effortHours: 4, layer: 'execution'})
CREATE (t2:Task {title: 'Implement React component', effortHours: 8, layer: 'execution'})
CREATE (p)-[:HAS_TASK]->(t1)
CREATE (p)-[:HAS_TASK]->(t2)
```

---

### Execution → Track (Layer 5 → 6)

**Relationships:**

- `OBSERVES` (Observation → Task/Project)

**Purpose:** Track execution progress and outcomes

**Example:**

```cypher
// Observation tracks Task completion
MATCH (t:Task {id: 'task_123'})
CREATE (obs:Observation {
  title: 'Task completed in 6 hours (estimated 8)',
  timestamp: '2025-11-05T14:00:00Z',
  layer: 'track'
})
CREATE (obs)-[:OBSERVES]->(t)
```

---

### Track → Mind (Layer 6 → 7)

**Relationships:**

- `INFORMS` (Observation → Insight/Pattern)

**Purpose:** Observations inform insights and pattern recognition

**Example:**

```cypher
// Observations inform Pattern
MATCH (obs:Observation)-[:OBSERVES]->(t:Task)
WHERE obs.title CONTAINS 'completed ahead of schedule'
CREATE (pat:Pattern {
  title: 'UI tasks consistently finish 20% faster than estimated',
  confidence: 0.75,
  sampleSize: 12,
  layer: 'mind'
})
CREATE (obs)-[:INFORMS]->(pat)
```

---

### Mind → All Layers (Layer 7 → 1-5)

**Relationships:**

- `APPLIED_TO` (Insight → Goal/Project)
- `TRIGGERS` (Insight → Decision)

**Purpose:** Apply learned insights back to improve execution

**Example:**

```cypher
// Insight applied to Goal
MATCH (ins:Insight {title: 'Morning decisions have 2x success rate'})
MATCH (g:Goal {title: 'Improve strategic decision quality'})
CREATE (ins)-[:APPLIED_TO]->(g)

// Insight triggers Decision
CREATE (d:Decision {
  title: 'Implement morning-only strategy sessions',
  rationale: 'Based on evening critic pattern analysis',
  layer: 'execution'
})
CREATE (ins)-[:TRIGGERS]->(d)
```

---

## Scalability & Why This Matters

### At Current Scale (1 user, 4 agents)

**Current State:**

- Single Neo4j database
- 4 agents running concurrently (Discuss, Decisions, Execute, Mind)
- Each agent writes to its own layers
- No write conflicts (layer ownership prevents collisions)

**Benefits:**

- ✅ No merge conflicts between agents
- ✅ Clear ownership and responsibility
- ✅ Parallel development without coordination overhead
- ✅ Agent failures isolated (Execute crash doesn't affect Discuss)

---

### At $500M Scale (10K+ users, 20+ agent types)

**Projected Scale:**

- 10,000+ concurrent users
- 20+ specialized agents (Support, Sales, Operations, Product, etc.)
- 1M+ nodes across 7 layers
- 100+ deployments/day

**Why Layer Architecture Scales:**

**1. Write Isolation:**

```
Support agents → Layer 6-7 (Track + Mind)
Sales agents → Layer 3 (Strategy - Objectives)
Operations agents → Layer 5 (Execution - Tasks)
Product agents → Layer 2 (Vision - Goals)
Constitutional agents → Layer 1 (Foundation)
```

No conflicts — each agent type writes to its own layer.

**2. Read Scalability:**

- All agents can READ all layers (via search)
- Neo4j read replicas handle read load (100K+ reads/sec)
- Write throughput limited only by layer owners (not global lock)

**3. Testing & Deployment:**

```
Layer 1-2 changes → Test Discuss agent only
Layer 3 changes → Test Decisions agent only
Layer 4-5 changes → Test Execute agent only
Layer 6-7 changes → Test Mind agent only
```

Independent testing = 4x faster CI/CD pipeline.

**4. Team Organization:**

```
Foundation Team → Layers 1-2 (constitutional graph)
Strategy Team → Layer 3 (OKRs, initiatives)
Execution Team → Layers 4-5 (projects, tasks)
Analytics Team → Layers 6-7 (metrics, insights)
```

Each team owns its layer, no coordination overhead.

---

## Common Anti-Patterns & How to Avoid

### ❌ Anti-Pattern 1: Discuss Agent Creating Tasks

**Wrong:**

```cypher
// Discuss agent creates Task directly (Layer 5)
CREATE (t:Task {title: 'Review financials', layer: 'execution'})
```

**Why Wrong:**

- Discuss owns Layers 1-2 (Foundation + Vision)
- Tasks are Layer 5 (Execute territory)
- Creates ownership ambiguity (who fixes broken tasks?)

**Correct:**

```cypher
// Discuss creates Goal (Layer 2)
CREATE (g:Goal {title: 'Complete Q4 financial review', layer: 'vision'})

// Handoff to Execute agent via protocol
EMIT handoff {from: 'discuss', to: 'execute', context: 'Goal created, needs task breakdown'}

// Execute agent creates Task under Goal
MATCH (g:Goal {title: 'Complete Q4 financial review'})
CREATE (t:Task {title: 'Review financials', layer: 'execution'})
CREATE (t)-[:SERVES]->(g)
```

---

### ❌ Anti-Pattern 2: Execute Agent Modifying Values

**Wrong:**

```cypher
// Execute agent modifies Value (Layer 1)
MATCH (v:Value {title: 'Autonomy'})
SET v.statement = 'Updated constitutional value'
```

**Why Wrong:**

- Execute owns Layers 4-5 (Tactics + Execution)
- Values are Layer 1 (Discuss territory)
- Constitutional changes require discussion, not execution

**Correct:**

```cypher
// Execute agent creates Observation (Layer 6)
CREATE (obs:Observation {
  title: 'Autonomy principle violated in Task execution flow',
  layer: 'track'
})

// Mind agent detects pattern
CREATE (pat:Pattern {
  title: 'Task approval flow conflicts with Autonomy value',
  layer: 'mind'
})
CREATE (obs)-[:INFORMS]->(pat)

// Handoff to Discuss agent
EMIT handoff {
  from: 'mind',
  to: 'discuss',
  context: 'Constitutional conflict detected, needs resolution'
}

// Discuss agent resolves (Layer 1 modification)
MATCH (v:Value {title: 'Autonomy'})
SET v.statement = 'Clarified after task flow analysis'
```

---

### ❌ Anti-Pattern 3: Decisions Agent Reading Execute Details Directly

**Wrong:**

```cypher
// Decisions agent queries all Tasks for ROI calculation
MATCH (t:Task)
WHERE t.status = 'completed'
RETURN sum(t.actualHours) AS totalHours
```

**Why Wrong:**

- Decisions owns Layer 3 (Strategy)
- Tasks are Layer 5 (Execute territory)
- Direct queries bypass layer boundaries and create tight coupling

**Correct:**

```cypher
// Decisions agent searches via hybrid search (respects layers)
CALL odei.neo4j.hybrid.search.v1({
  query: 'completed tasks ROI analysis',
  layers: ['execution'],  // Explicitly specify allowed layers
  topK: 50
})
YIELD node
RETURN node

// Or: Use Mind agent's aggregated Pattern/Insight
MATCH (pat:Pattern {title: 'ROI by task type'})-[:INFORMS]->(ins:Insight)
WHERE ins.layer = 'mind'
RETURN ins.data
```

---

### ❌ Anti-Pattern 4: Mind Agent Triggering Changes Directly

**Wrong:**

```cypher
// Mind agent modifies Goal based on pattern
MATCH (g:Goal {id: 'goal_q4'})
SET g.status = 'deprioritized'  // Direct modification
```

**Why Wrong:**

- Mind owns Layers 6-7 (Track + Mind)
- Goals are Layer 2 (Discuss territory)
- Mind is read-only observer, cannot modify execution

**Correct:**

```cypher
// Mind agent creates Insight (Layer 7)
CREATE (ins:Insight {
  title: 'Goal completion rate 65% suggests scope too large',
  recommendation: 'Reduce goal scope by 30%',
  layer: 'mind'
})

// Insight applied to Goal (relationship, not modification)
MATCH (g:Goal {id: 'goal_q4'})
CREATE (ins)-[:APPLIED_TO]->(g)

// Handoff to Discuss agent
EMIT handoff {
  from: 'mind',
  to: 'discuss',
  context: 'Insight suggests goal scope reduction'
}

// Discuss agent decides whether to act (Layer 2 modification)
MATCH (g:Goal {id: 'goal_q4'})
SET g.scope = 'reduced',
    g.rationale = 'Based on Mind agent analysis'
```

---

## Layer Access Control Matrix

| Agent         | Layer 1 (Foundation)  | Layer 2 (Vision)      | Layer 3 (Strategy)    | Layer 4 (Tactics)     | Layer 5 (Execution)   | Layer 6 (Track) | Layer 7 (Mind)  |
| ------------- | --------------------- | --------------------- | --------------------- | --------------------- | --------------------- | --------------- | --------------- |
| **Discuss**   | ✅ Read/Write         | ✅ Read/Write         | 🔍 Search only        | 🔍 Search only        | 📊 Summary only       | 📊 Summary only | 📊 Summary only |
| **Decisions** | 🔍 Search only        | 🔍 Search only        | ✅ Read/Write         | 🔍 Search only        | 🔍 Search only        | 📊 Summary only | 📊 Summary only |
| **Execute**   | 🔍 Search only        | 🔍 Search only        | 🔍 Search only        | ✅ Read/Write         | ✅ Read/Write         | 📊 Summary only | 📊 Summary only |
| **Mind**      | 🔍 Search (read-only) | 🔍 Search (read-only) | 🔍 Search (read-only) | 🔍 Search (read-only) | 🔍 Search (read-only) | ✅ Read/Write   | ✅ Read/Write   |

**Legend:**

- ✅ **Read/Write** — Full CRUD access (create, read, update, delete)
- 🔍 **Search only** — Can search/read nodes via `odei.neo4j.hybrid.search.v1`, cannot modify
- 📊 **Summary only** — Can access aggregated summary data (e.g., `workload.assess.v1`), no direct queries

---

## Implementation in MCP Configuration

### Discuss Agent `.mcp.json`

```json
{
  "mcpServers": {
    "odei-neo4j": {
      "command": "node",
      "args": ["/Users/ai/ODEI/servers/odei-neo4j/dist/index.js"],
      "env": {
        "AGENT_ROLE": "discuss",
        "ALLOWED_LAYERS": "foundation,vision"
      },
      "toolAllowList": [
        "odei.neo4j.value.create.v1",
        "odei.neo4j.value.update.v1",
        "odei.neo4j.principle.create.v1",
        "odei.neo4j.principle.update.v1",
        "odei.neo4j.guardrail.create.v1",
        "odei.neo4j.vision.create.v1",
        "odei.neo4j.goal.create.v1",
        "odei.neo4j.goal.update.v1",
        "odei.neo4j.hybrid.search.v1",
        "odei.neo4j.hybrid.plusplus.search.v1",
        "odei.neo4j.foundation.list.v2",
        "odei.neo4j.vision.list.v2",
        "odei.neo4j.workload.assess.v1"
      ]
    }
  }
}
```

### Execute Agent `.mcp.json`

```json
{
  "mcpServers": {
    "odei-neo4j": {
      "command": "node",
      "args": ["/Users/ai/ODEI/servers/odei-neo4j/dist/index.js"],
      "env": {
        "AGENT_ROLE": "execute",
        "ALLOWED_LAYERS": "tactics,execution"
      },
      "toolAllowList": [
        "odei.neo4j.project.create.v1",
        "odei.neo4j.project.update.v1",
        "odei.neo4j.task.create.v1",
        "odei.neo4j.task.update.v1",
        "odei.neo4j.timeblock.create.v1",
        "odei.neo4j.hybrid.search.v1",
        "odei.neo4j.workload.assess.v1"
      ]
    }
  }
}
```

---

## Testing Layer Boundaries

### Test: Discuss Cannot Create Tasks

```bash
# Run from Discuss agent workspace
cd /Users/ai/ODEI/agents/discuss

# Attempt to create Task (should fail)
echo "Testing layer boundary enforcement..."

# Expected error: "Agent 'discuss' cannot write to layer 'execution'"
```

**Expected Behavior:**

- MCP server receives `odei.neo4j.task.create.v1` request
- Checks `AGENT_ROLE=discuss` from environment
- Checks `ALLOWED_LAYERS=foundation,vision`
- Rejects request: "Task is Layer 5 (execution), agent only allowed foundation,vision"

---

### Test: Execute Cannot Modify Values

```bash
# Run from Execute agent workspace
cd /Users/ai/ODEI/agents/execute

# Attempt to modify Value (should fail)
echo "Testing layer boundary enforcement..."

# Expected error: "Agent 'execute' cannot write to layer 'foundation'"
```

---

## Migration Path from Old Architecture

**Before (Agent-Specific Servers):**

```
discuss-neo4j → discuss_hybrid_search, discuss_value_create
decisions-neo4j → decisions_hybrid_search, decisions_objective_create
execute-neo4j → execute_hybrid_search, execute_task_create
```

**After (Unified Server + Layer Enforcement):**

```
odei-neo4j (shared) → odei.neo4j.hybrid.search.v1 (all agents)
                   → odei.neo4j.value.create.v1 (Discuss only)
                   → odei.neo4j.objective.create.v1 (Decisions only)
                   → odei.neo4j.task.create.v1 (Execute only)
```

**Migration Steps:**

1. ✅ Delete agent-specific Neo4j servers (Nov 5, 2025)
2. ✅ Update all agent configs to use `odei-neo4j` shared server
3. ✅ Add `AGENT_ROLE` environment variable to each agent config
4. ⏳ Implement server-side layer enforcement middleware (Week of Nov 6)
5. ⏳ Add test suite validating layer boundaries (Week of Nov 6)

---

## References

- **Relationship definitions:** `/Users/ai/ODEI/servers/odei-neo4j/src/domain/relationships.ts`
- **Architecture analysis:** `/Users/ai/ODEI/ARCHITECTURE_ANALYSIS.md`
- **MCP consolidation plan:** `/Users/ai/ODEI/MCP_CONSOLIDATION_ANALYSIS.md`
- **Discuss agent scope:** `/Users/ai/ODEI/agents/discuss/.claude/CLAUDE.md`

---

## Changelog

**v1.0 (Nov 5, 2025):**

- Initial documentation of 7-layer architecture
- Defined layer ownership by agents
- Documented cross-layer relationship patterns
- Established anti-patterns and testing guidelines

---

**Document Owner:** Architecture Team
**Next Review:** December 1, 2025
**Status:** ✅ Active
