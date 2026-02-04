# ODEI Relationship Reference

**Version:** 1.0
**Generated:** 2025-12-25
**Total Relationships:** 67

---

## Table of Contents

1. [Relationship Types Overview](#relationship-types-overview)
2. [Constitutional Relationships (Foundation)](#constitutional-relationships-foundation)
3. [Directional Relationships (Vision → Strategy → Execution)](#directional-relationships-vision--strategy--execution)
4. [Learning Relationships (Track → Mind → Foundation)](#learning-relationships-track--mind--foundation)
5. [Time & Capacity Relationships](#time--capacity-relationships)
6. [Relationship Properties](#relationship-properties)
7. [Layer Crossing Rules](#layer-crossing-rules)

---

## Relationship Types Overview

ODEI uses **67 relationship types** organized by semantic purpose:

| Category | Count | Examples |
|----------|-------|----------|
| Constitutional | 11 | HOLDS_VALUE, FOLLOWS_PRINCIPLE, ALIGNS_WITH |
| Strategic | 15 | SERVES, ADVANCES, CONTAINS, HAS_KEY_RESULT |
| Tactical | 9 | HAS_PROJECT, HAS_TASK, DEPENDS_ON, USES |
| Temporal | 5 | SCHEDULED_AS, LOGGED_AS, GENERATES_TASK |
| Learning | 12 | INFORMS, VALIDATES, DEMONSTRATES, TRIGGERS |
| Observational | 7 | TRACKS, OBSERVES, APPLIED_TO |
| Organizational | 8 | ENABLES, EXECUTES, DELIVERS, SUPPORTS |

---

## Constitutional Relationships (Foundation)

### HOLDS_VALUE
**From:** Human (foundation)
**To:** Value (foundation)
**Direction:** Directional
**Purpose:** Indicates that a human holds or embodies a specific value

**Example:**
```cypher
MATCH (h:Human {title: 'Anton Illarionov'})
MATCH (v:Value {title: 'Financial Freedom'})
MERGE (h)-[:HOLDS_VALUE]->(v)
```

---

### FOLLOWS_PRINCIPLE
**From:** Human (foundation)
**To:** Principle (foundation)
**Direction:** Directional
**Purpose:** Indicates that a human follows or adheres to a specific principle

**Example:**
```cypher
MATCH (h:Human {title: 'Anton Illarionov'})
MATCH (p:Principle {title: 'AI Principal Primacy'})
MERGE (h)-[:FOLLOWS_PRINCIPLE]->(p)
```

---

### RESPECTS_GUARDRAIL
**From:** Human (foundation)
**To:** Guardrail (foundation)
**Direction:** Directional
**Purpose:** Indicates that a human respects or operates within a specific guardrail

**Example:**
```cypher
MATCH (h:Human {title: 'Anton Illarionov'})
MATCH (g:Guardrail {title: 'No Evening Strategic Pivots'})
MERGE (h)-[:RESPECTS_GUARDRAIL]->(g)
```

---

### SUPPORTED_BY (Value → Principle)
**From:** Value (foundation)
**To:** Principle (foundation)
**Direction:** Directional
**Properties:** `weight` (required), `notes` (optional)
**Purpose:** Shows how principles operationalize values

**Example:**
```cypher
MATCH (v:Value {title: 'Autonomy'})
MATCH (p:Principle {title: 'Make reversible decisions quickly'})
MERGE (v)-[:SUPPORTED_BY {weight: 0.9}]->(p)
```

---

### ENFORCED_BY
**From:** Principle (foundation)
**To:** Guardrail (foundation)
**Direction:** Directional
**Purpose:** Shows how guardrails protect principles

**Example:**
```cypher
MATCH (p:Principle {title: 'Family First'})
MATCH (g:Guardrail {title: 'No Weekend Commitments'})
MERGE (p)-[:ENFORCED_BY]->(g)
```

---

### ALIGNS_WITH (Vision → Foundation)
**From:** Value/Principle (foundation), Goal (vision), Business (vision)
**To:** Vision/Goal (vision), Value/Principle (foundation)
**Direction:** Bidirectional (depends on context)
**Properties:** `weight` (required), `notes` (optional)
**Purpose:** Ensures constitutional alignment between layers

**Examples:**
```cypher
// Goal aligns with Value
MATCH (g:Goal {title: '$500M Net Worth by 2035'})
MATCH (v:Value {title: 'Financial Freedom'})
MERGE (g)-[:ALIGNS_WITH {weight: 1.0}]->(v)

// Business aligns with Principle
MATCH (b:Business {title: 'Tipz CBS'})
MATCH (p:Principle {title: 'Excellence Over Mediocrity'})
MERGE (b)-[:ALIGNS_WITH {weight: 0.85}]->(p)
```

---

### PARTNERS_WITH
**From:** Human (foundation)
**To:** AI (foundation), Partnership (foundation)
**Direction:** Directional
**Purpose:** Defines critical collaboration relationships

**Example:**
```cypher
MATCH (h:Human {title: 'Anton Illarionov'})
MATCH (ai:AI {title: 'Claude (AI Principal)'})
MERGE (h)-[:PARTNERS_WITH]->(ai)
```

---

### PROTECTED_BY
**From:** Partnership (foundation)
**To:** Guardrail (foundation)
**Direction:** Directional
**Purpose:** Links partnerships to guardrails that protect them

**Example:**
```cypher
MATCH (p:Partnership {title: 'Nisanov Family Office'})
MATCH (g:Guardrail {title: 'No Partnerships by Default'})
MERGE (p)-[:PROTECTED_BY]->(g)
```

---

### DEFINED_BY
**From:** Human (foundation)
**To:** Core (foundation)
**Direction:** Directional
**Purpose:** Connects humans to core identity pillars

**Example:**
```cypher
MATCH (h:Human {title: 'Anton Illarionov'})
MATCH (c:Core {title: 'Solo+AI Operating System'})
MERGE (h)-[:DEFINED_BY]->(c)
```

---

### RELATES_TO
**From:** Relationship (foundation)
**To:** Partnership (foundation), Value (foundation)
**Direction:** Directional
**Purpose:** Connects relationship records to partnerships or values

**Example:**
```cypher
MATCH (r:Relationship {title: 'Nisanov Family History'})
MATCH (p:Partnership {title: 'Nisanov Family Office'})
MERGE (r)-[:RELATES_TO]->(p)
```

---

### OPERATES_IN
**From:** Human (foundation)
**To:** Context (foundation)
**Direction:** Directional
**Purpose:** Indicates that a human operates within a specific context

**Example:**
```cypher
MATCH (h:Human {title: 'Anton Illarionov'})
MATCH (c:Context {title: 'Parenthood'})
MERGE (h)-[:OPERATES_IN]->(c)
```

---

## Directional Relationships (Vision → Strategy → Execution)

### SERVES (Multi-Layer)

**Pattern 1: Goal → Vision**
**From:** Goal (vision)
**To:** Vision (vision)
**Purpose:** Shows how specific goals contribute to overarching vision

```cypher
MATCH (g:Goal {title: '$500M Net Worth by 2035'})
MATCH (v:Vision {title: 'Generational Financial Freedom'})
MERGE (g)-[:SERVES]->(v)
```

**Pattern 2: Objective → Goal**
**From:** Objective (strategy)
**To:** Goal (vision)
**Purpose:** Connects strategic objectives to vision goals

```cypher
MATCH (o:Objective {title: 'Build $500M Equity'})
MATCH (g:Goal {title: '$500M Net Worth by 2035'})
MERGE (o)-[:SERVES]->(g)
```

**Pattern 3: Strategy → Goal**
**From:** Strategy (strategy)
**To:** Goal/Vision (vision)
**Properties:** `weight` (required), `notes` (optional)
**Purpose:** Connects multi-quarter strategy campaigns to goals

```cypher
MATCH (s:Strategy {title: 'CBS Q4 2025 - Q1 2026'})
MATCH (g:Goal {title: 'CBS Strategy Package Complete'})
MERGE (s)-[:SERVES {weight: 1.0}]->(g)
```

**Pattern 4: Task → Goal (Direct Traceability)**
**From:** Decision/Task (execution)
**To:** Goal (vision)
**Purpose:** Direct execution-to-vision connection

```cypher
MATCH (t:Task {title: 'Implement Auth Endpoint'})
MATCH (g:Goal {title: 'Launch Tipz CBS MVP'})
MERGE (t)-[:SERVES]->(g)
```

**Pattern 5: Decision → Project**
**From:** Decision (execution)
**To:** Project (tactics)
**Purpose:** Shows how decisions impact project execution

```cypher
MATCH (d:Decision {title: 'Hire First Engineer'})
MATCH (p:Project {title: 'Tipz Integration API'})
MERGE (d)-[:SERVES]->(p)
```

---

### ADVANCES

**Pattern 1: Initiative → Objective**
**From:** Initiative (strategy)
**To:** Objective (strategy)
**Purpose:** Shows how initiatives drive objective completion

```cypher
MATCH (i:Initiative {title: 'Tipz Partnership'})
MATCH (o:Objective {title: 'Increase MRR to $500K'})
MERGE (i)-[:ADVANCES]->(o)
```

**Pattern 2: Decision → Objective/KeyResult/Initiative**
**From:** Decision (execution)
**To:** Objective/KeyResult/Initiative (strategy)
**Purpose:** Bottom-up traceability from decisions to strategy

```cypher
MATCH (d:Decision {title: 'Hire First Engineer'})
MATCH (o:Objective {title: 'Increase MRR to $500K'})
MERGE (d)-[:ADVANCES]->(o)
```

---

### CONTAINS
**From:** Strategy (strategy)
**To:** Objective/KeyResult/Initiative/Risk/Milestone (strategy)
**Purpose:** Organizes OKR elements under strategy container

**Example:**
```cypher
MATCH (s:Strategy {title: 'CBS Q4 2025 - Q1 2026'})
MATCH (o:Objective {title: 'Build $500M Equity'})
MERGE (s)-[:CONTAINS]->(o)
```

---

### HAS_KEY_RESULT
**From:** Objective (strategy)
**To:** KeyResult (strategy)
**Purpose:** Implements OKR framework (Objectives have Key Results)

**Example:**
```cypher
MATCH (o:Objective {title: 'Increase MRR to $500K'})
MATCH (kr:KeyResult {title: 'Reach 100 Paying Customers'})
MERGE (o)-[:HAS_KEY_RESULT]->(kr)
```

---

### MEASURES (Inverse of HAS_KEY_RESULT)
**From:** KeyResult (strategy)
**To:** Objective (strategy)
**Purpose:** Inverse relationship for bidirectional queries

**Example:**
```cypher
MATCH (kr:KeyResult {title: 'Reach 100 Paying Customers'})
MATCH (o:Objective {title: 'Increase MRR to $500K'})
MERGE (kr)-[:MEASURES]->(o)
```

---

### HAS_MILESTONE
**From:** Initiative (strategy)
**To:** Milestone (strategy)
**Properties:** `criticality`, `notes`, `sequence`, `weight` (all optional)
**Purpose:** Enforces 3-5 milestone rule for every initiative

**Example:**
```cypher
MATCH (i:Initiative {title: 'Tipz Partnership'})
MATCH (m:Milestone {title: 'Dec 6 Strategy Complete'})
MERGE (i)-[:HAS_MILESTONE {sequence: 1, criticality: 'high'}]->(m)
```

---

### DEPENDS_ON (Milestone Sequencing)
**From:** Milestone (strategy)
**To:** Milestone (strategy)
**Properties:** `dependencyType`, `lagDays`, `notes`, `weight` (all optional)
**Purpose:** Maps milestone sequencing (M1 → M2 → M3)

**Example:**
```cypher
MATCH (m1:Milestone {title: 'Q1 POC Complete'})
MATCH (m2:Milestone {title: 'Dec 6 Strategy Complete'})
MERGE (m1)-[:DEPENDS_ON {dependencyType: 'finish-to-start', lagDays: 0}]->(m2)
```

---

### DEPENDS_ON (Project Dependencies)
**From:** Project (tactics)
**To:** Project (tactics)
**Properties:** Same as milestone dependencies
**Purpose:** Creates project critical path

**Example:**
```cypher
MATCH (p1:Project {title: 'Tipz Public Launch'})
MATCH (p2:Project {title: 'Tipz Integration API'})
MERGE (p1)-[:DEPENDS_ON {dependencyType: 'finish-to-start'}]->(p2)
```

---

### HAS_GOAL (Season/Business → Goal)
**From:** Season (vision), Business (vision)
**To:** Goal (vision)
**Purpose:** Connects time-bounded seasons or businesses to their goals

**Examples:**
```cypher
// Season has Goal
MATCH (s:Season {title: '2024 Q1'})
MATCH (g:Goal {title: 'Launch Tipz'})
MERGE (s)-[:HAS_GOAL]->(g)

// Business has Goal
MATCH (b:Business {title: 'Automated Exchange'})
MATCH (g:Goal {title: 'BestChange Listing'})
MERGE (b)-[:HAS_GOAL]->(g)
```

---

### HAS_PROJECT
**From:** Initiative (strategy), Area (tactics), Goal (vision)
**To:** Project (tactics)
**Purpose:** Bridges layers (strategic/vision → tactical)

**Examples:**
```cypher
// Initiative → Project
MATCH (i:Initiative {title: 'Tipz Partnership'})
MATCH (p:Project {title: 'Tipz Integration API'})
MERGE (i)-[:HAS_PROJECT]->(p)

// Area → Project
MATCH (a:Area {title: 'Business Development'})
MATCH (p:Project {title: 'Tipz Dec 6 Pitch'})
MERGE (a)-[:HAS_PROJECT]->(p)

// Goal → Project (direct)
MATCH (g:Goal {title: 'Launch Tipz'})
MATCH (p:Project {title: 'Tipz Integration API'})
MERGE (g)-[:HAS_PROJECT]->(p)
```

---

### HAS_TASK
**From:** Project (tactics)
**To:** Task (execution)
**Purpose:** Decomposes projects into actionable tasks

**Example:**
```cypher
MATCH (p:Project {title: 'Tipz Integration API'})
MATCH (t:Task {title: 'Implement Auth Endpoint'})
MERGE (p)-[:HAS_TASK]->(t)
```

---

### DELIVERS
**From:** Project (tactics)
**To:** Strategy (strategy)
**Properties:** `deliveryDate`, `notes`, `weight` (all optional)
**Purpose:** Links project execution to strategy campaigns

**Example:**
```cypher
MATCH (p:Project {title: 'CBS Strategy Doc'})
MATCH (s:Strategy {title: 'CBS Q4 2025 - Q1 2026'})
MERGE (p)-[:DELIVERS {deliveryDate: '2025-12-06'}]->(s)
```

---

### EXECUTES
**From:** Initiative (strategy)
**To:** Business (vision)
**Properties:** `notes`, `phase`, `primaryInitiative`, `weight` (all optional)
**Purpose:** Connects initiatives to the businesses they operationalize

**Example:**
```cypher
MATCH (i:Initiative {title: 'Tipz Partnership'})
MATCH (b:Business {title: 'Tipz CBS'})
MERGE (i)-[:EXECUTES {phase: 'MVP', primaryInitiative: true}]->(b)
```

---

### ENABLES
**From:** Partnership (foundation), Vision (vision)
**To:** Vision/Goal/Business (vision)
**Properties:** `activation`, `criticality`, `mechanism`, `notes`, `weight` (all optional)
**Purpose:** Shows how partnerships or visions enable execution

**Examples:**
```cypher
// Partnership enables Business
MATCH (p:Partnership {title: 'Tipz.ae Co-Founder'})
MATCH (b:Business {title: 'Tipz CBS'})
MERGE (p)-[:ENABLES {criticality: 'high', mechanism: '7% equity, CPO scope'}]->(b)

// Vision enables Business
MATCH (v:Vision {title: 'Generational Freedom'})
MATCH (b:Business {title: 'Tipz CBS'})
MERGE (v)-[:ENABLES]->(b)
```

---

### PURSUES_GOAL
**From:** Human (foundation)
**To:** Goal (vision)
**Purpose:** Creates personal accountability and ownership

**Example:**
```cypher
MATCH (h:Human {title: 'Anton Illarionov'})
MATCH (g:Goal {title: '$500M Net Worth by 2035'})
MERGE (h)-[:PURSUES_GOAL]->(g)
```

---

### SUPPORTED_BY (Goal Dependencies)
**From:** Goal (vision)
**To:** Goal (vision)
**Properties:** `weight` (required), `notes` (optional)
**Purpose:** Creates goal hierarchies and dependencies

**Example:**
```cypher
MATCH (g1:Goal {title: 'Scale to 1000 Users'})
MATCH (g2:Goal {title: 'Launch MVP'})
MERGE (g1)-[:SUPPORTED_BY {weight: 1.0}]->(g2)
```

---

### SUPPORTS (Non-Blocking Project Synergy)
**From:** Project (tactics)
**To:** Project (tactics)
**Purpose:** Shows complementary work (not blocking)

**Example:**
```cypher
MATCH (p1:Project {title: 'Marketing Campaign'})
MATCH (p2:Project {title: 'Tipz Public Launch'})
MERGE (p1)-[:SUPPORTS]->(p2)
```

---

## Learning Relationships (Track → Mind → Foundation)

### TRACKS
**From:** Observation (track)
**To:** Metric (track)
**Purpose:** Connects observations to what they measure

**Example:**
```cypher
MATCH (obs:Observation {title: 'MRR March 2024'})
MATCH (m:Metric {title: 'Monthly Recurring Revenue'})
MERGE (obs)-[:TRACKS]->(m)
```

---

### OBSERVES
**From:** Observation (track)
**To:** Goal (vision), Project (tactics), Business (vision)
**Purpose:** Tracks progress on goals, projects, or businesses

**Examples:**
```cypher
// Observe Goal
MATCH (obs:Observation {title: 'Net Worth Q1 2024'})
MATCH (g:Goal {title: '$500M Net Worth by 2035'})
MERGE (obs)-[:OBSERVES]->(g)

// Observe Project
MATCH (obs:Observation {title: 'API Integration Velocity'})
MATCH (p:Project {title: 'Tipz Integration API'})
MERGE (obs)-[:OBSERVES]->(p)

// Observe Business
MATCH (obs:Observation {title: 'Tipz Valuation Jan 2026'})
MATCH (b:Business {title: 'Tipz CBS'})
MERGE (obs)-[:OBSERVES]->(b)
```

---

### INFORMS
**From:** Observation (track)
**To:** Insight/Pattern (mind)
**Properties:** `notes`, `weight` (both optional)
**Purpose:** Shows how observations lead to learning

**Example:**
```cypher
MATCH (obs:Observation {title: 'MRR Growth Feb 2024'})
MATCH (ins:Insight {title: 'Partnership Revenue Driver'})
MERGE (obs)-[:INFORMS {weight: 0.85}]->(ins)
```

---

### VALIDATES
**From:** Evidence (mind)
**To:** Pattern/Insight (mind)
**Properties:** `notes`, `strength`, `weight` (all optional)
**Purpose:** Connects evidence to patterns/insights they prove

**Example:**
```cypher
MATCH (e:Evidence {title: 'Aegis.im Exit'})
MATCH (p:Pattern {title: 'Solo+AI Delivers'})
MERGE (e)-[:VALIDATES {strength: 'strong', weight: 0.95}]->(p)
```

---

### DEMONSTRATES
**From:** Pattern (mind)
**To:** Value/Principle (foundation)
**Properties:** `weight` (required), `notes` (optional)
**Purpose:** Shows how patterns prove values in action

**Example:**
```cypher
MATCH (p:Pattern {title: 'Solo+AI Delivers'})
MATCH (v:Value {title: 'Human-AI Symbiosis'})
MERGE (p)-[:DEMONSTRATES {weight: 0.9}]->(v)
```

---

### APPLIED_TO
**From:** Insight (mind)
**To:** Goal (vision), Project (tactics)
**Purpose:** Shows how learning influences future execution

**Examples:**
```cypher
// Insight → Goal
MATCH (ins:Insight {title: 'Partnership Revenue Driver'})
MATCH (g:Goal {title: '$500M Net Worth by 2035'})
MERGE (ins)-[:APPLIED_TO]->(g)

// Insight → Project
MATCH (ins:Insight {title: 'API Complexity Underestimated'})
MATCH (p:Project {title: 'Tipz Integration API'})
MERGE (ins)-[:APPLIED_TO]->(p)
```

---

### TRIGGERS
**From:** Insight (mind)
**To:** Decision (execution)
**Purpose:** Shows how insights drive action

**Example:**
```cypher
MATCH (ins:Insight {title: 'Engineering Bottleneck'})
MATCH (d:Decision {title: 'Hire First Engineer'})
MERGE (ins)-[:TRIGGERS]->(d)
```

---

### SUPPORTED_BY (Decision ← Evidence)
**From:** Decision (execution)
**To:** Evidence (mind)
**Properties:** `notes`, `strength`, `weight` (all optional)
**Purpose:** Evidence-based decision making

**Example:**
```cypher
MATCH (d:Decision {title: 'Hire First Engineer'})
MATCH (e:Evidence {title: 'Velocity Bottleneck Analysis'})
MERGE (d)-[:SUPPORTED_BY {strength: 'high', weight: 0.9}]->(e)
```

---

### AGGREGATES
**From:** Note (mind)
**To:** Evidence/Pattern/Insight/Observation/Experience/Source (mind)
**Properties:** `notes`, `weight` (both optional)
**Purpose:** Note hubs organize institutional memory

**Example:**
```cypher
MATCH (n:Note {title: 'Institutional Memory'})
MATCH (e:Evidence {title: 'Aegis Exit'})
MERGE (n)-[:AGGREGATES {weight: 1.0}]->(e)
```

---

### INFORMS_IDENTITY
**From:** Note (mind)
**To:** Human (foundation)
**Properties:** `notes`, `weight` (both optional)
**Purpose:** Mind layer informs human identity

**Example:**
```cypher
MATCH (n:Note {title: 'Institutional Memory'})
MATCH (h:Human {title: 'Anton Illarionov'})
MERGE (n)-[:INFORMS_IDENTITY {weight: 1.0}]->(h)
```

---

### EXHIBITS
**From:** Human (foundation)
**To:** Pattern (mind)
**Properties:** `notes`, `weight` (both optional)
**Purpose:** Connects people to behavioral patterns

**Example:**
```cypher
MATCH (h:Human {title: 'Anton Illarionov'})
MATCH (p:Pattern {title: 'Solo+AI Delivers'})
MERGE (h)-[:EXHIBITS {weight: 0.95}]->(p)
```

---

### SHAPED_BY
**From:** Human (foundation)
**To:** Evidence/Experience/Observation (mind)
**Purpose:** Formative experiences that built skills/perspectives

**Example:**
```cypher
MATCH (h:Human {title: 'Anton Illarionov'})
MATCH (e:Evidence {title: 'FixedFloat Exit'})
MERGE (h)-[:SHAPED_BY]->(e)
```

---

## Time & Capacity Relationships

### SCHEDULED_AS
**From:** Task (execution)
**To:** TimeBlock (execution)
**Properties:** `calendarProvider`, `confirmed`, `notes`, `weight` (all optional)
**Purpose:** Connects tasks to calendar time

**Example:**
```cypher
MATCH (t:Task {title: 'Implement Auth Endpoint'})
MATCH (tb:TimeBlock {start: '2024-03-15T09:00:00Z'})
MERGE (t)-[:SCHEDULED_AS {confirmed: true, calendarProvider: 'apple'}]->(tb)
```

---

### LOGGED_AS
**From:** Task (execution)
**To:** WorkSession (execution)
**Properties:** `notes`, `weight` (both optional)
**Purpose:** Tracks actual work completed

**Example:**
```cypher
MATCH (t:Task {title: 'Implement Auth Endpoint'})
MATCH (ws:WorkSession {start: '2024-03-15T09:00:00Z'})
MERGE (t)-[:LOGGED_AS {weight: 1.0}]->(ws)
```

---

### GENERATES_TASK
**From:** Process (tactics)
**To:** Task (execution)
**Properties:** `lastGenerated`, `nextDue`, `notes`, `recurrenceRule`, `weight` (all optional)
**Purpose:** Recurring processes spawn task instances

**Example:**
```cypher
MATCH (pr:Process {title: 'MMA Training 3x/week'})
MATCH (t:Task {title: 'MMA Monday Nov 13'})
MERGE (pr)-[:GENERATES_TASK {
  recurrenceRule: 'FREQ=WEEKLY;BYDAY=MO,WE,FR',
  lastGenerated: '2024-11-13T12:00:00Z'
}]->(t)
```

---

### HAS_PROCESS
**From:** Area (tactics)
**To:** Process (tactics)
**Purpose:** Maps recurring processes to life domains

**Example:**
```cypher
MATCH (a:Area {title: 'Health & Performance'})
MATCH (pr:Process {title: 'MMA Training 3x/week'})
MERGE (a)-[:HAS_PROCESS]->(pr)
```

---

### USES (Project/Area/Process → System)
**From:** Project/Area/Process (tactics)
**To:** System (tactics)
**Properties:** `cost`, `criticality`, `notes`, `since`, `weight` (all optional)
**Purpose:** Tracks infrastructure dependencies

**Example:**
```cypher
MATCH (p:Project {title: 'Tipz Strategy Package'})
MATCH (s:System {title: 'Miro Visualization'})
MERGE (p)-[:USES {criticality: 'high', since: '2024-11-01'}]->(s)
```

---

### USES (Business → System)
**From:** Business (vision)
**To:** System (tactics)
**Properties:** Same as above
**Purpose:** Business-level infrastructure dependencies

**Example:**
```cypher
MATCH (b:Business {title: 'Automated Exchange'})
MATCH (s:System {title: 'Sakartvelo Infrastructure'})
MERGE (b)-[:USES {criticality: 'high', cost: '$5000/mo'}]->(s)
```

---

## Relationship Properties

### Common Properties

Many relationships support these optional properties:

| Property | Type | Description |
|----------|------|-------------|
| `weight` | number | Importance/strength (0.0-1.0) |
| `notes` | string | Context/explanation |
| `criticality` | enum | `"low"`, `"medium"`, `"high"` |

### Specialized Properties

#### Schedule Properties
- `calendarProvider` — `"apple"`, `"google"`, etc.
- `confirmed` — Boolean
- `recurrenceRule` — iCal RRULE format

#### Dependency Properties
- `dependencyType` — `"finish-to-start"`, `"start-to-start"`, etc.
- `lagDays` — Number of days delay

#### Evidence Properties
- `strength` — `"weak"`, `"medium"`, `"strong"`
- `sourceRef` — URI/reference to source

#### Financial Properties
- `cost` — Dollar amount string (e.g., `"$5000/mo"`)
- `since` — ISO date when relationship started

---

## Layer Crossing Rules

### Allowed Cross-Layer Patterns

✅ **Upward References** (Lower → Higher)
```cypher
// Execution → Vision
Task -[:SERVES]-> Goal  // OK

// Track → Mind
Observation -[:INFORMS]-> Insight  // OK

// Strategy → Vision
Objective -[:SERVES]-> Goal  // OK
```

✅ **Downward Decomposition** (Higher → Lower)
```cypher
// Vision → Tactics
Goal -[:HAS_PROJECT]-> Project  // OK

// Strategy → Tactics
Initiative -[:HAS_PROJECT]-> Project  // OK

// Tactics → Execution
Project -[:HAS_TASK]-> Task  // OK
```

✅ **Observation Patterns** (Track can observe any layer)
```cypher
Observation -[:OBSERVES]-> Goal  // Vision
Observation -[:OBSERVES]-> Project  // Tactics
Observation -[:OBSERVES]-> Business  // Vision
```

✅ **Learning Feedback** (Mind can inform any layer)
```cypher
Insight -[:APPLIED_TO]-> Goal  // Vision
Insight -[:APPLIED_TO]-> Project  // Tactics
Pattern -[:DEMONSTRATES]-> Value  // Foundation
```

### Forbidden Cross-Layer Patterns

❌ **Foundation Cannot Reference Lower Layers**
```cypher
Value -[:DEPENDS_ON]-> Task  // INVALID
Principle -[:HAS_PROJECT]-> Project  // INVALID
```

❌ **Execution Cannot Modify Foundation**
```cypher
Decision -[:MODIFIES]-> Value  // INVALID
Task -[:OVERRIDES]-> Principle  // INVALID
```

❌ **Same-Layer Modifications Across Contexts**
```cypher
// Task in one project cannot modify task in another without intermediate
Task -[:MODIFIES]-> Task  // Use DEPENDS_ON instead
```

---

**Next:** See [GRAPH-MODEL.md](./GRAPH-MODEL.md) for visual architecture and [QUERY-COOKBOOK.md](./QUERY-COOKBOOK.md) for usage examples.
