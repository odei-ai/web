# ODEI Graph Model — Visual Architecture

**Version:** 1.0
**Generated:** 2025-12-25
**Auto-generated from:** `.claude/SCHEMA.md` (schema version 1.3.0)

---

## Table of Contents

1. [Overview](#overview)
2. [7-Layer Visual Model](#7-layer-visual-model)
3. [Layer-by-Layer Breakdown](#layer-by-layer-breakdown)
4. [Node Type Reference](#node-type-reference)
5. [Relationship Flow Patterns](#relationship-flow-patterns)
6. [Cross-Layer Connections](#cross-layer-connections)

---

## Overview

The ODEI Memory Atlas is a **constitutional graph database** organized into 7 hierarchical layers. Each layer serves a specific purpose in the knowledge hierarchy, from foundational identity (Layer 1) to learned insights (Layer 7).

**Key Statistics:**
- **38 node types** across 7 layers
- **67 relationship types** with strict layer constraints
- **4 modules** with role-based access control (RBAC)
- **Provenance required** on all writes (`module`, `actor`, `source`, `confidence`)

**Design Principles:**
1. **Constitutional by Design** — Layer boundaries enforce separation of concerns
2. **Upward Traceability** — Everything traces back to Foundation (values, principles)
3. **Read-Only Observer Pattern** — Higher layers inform lower layers without modifying them
4. **Module Ownership** — Each module owns specific layers for write operations

---

## 7-Layer Visual Model

```
┌────────────────────────────────────────────────────────────────────┐
│  LAYER 7: MIND (Learning & Pattern Recognition)                   │
│  ┌──────────────────────────────────────────────────────────────┐ │
│  │  Insight (5)    Pattern (5)    Evidence (5)                  │ │
│  │  Note (hub)     Source (URIs)                                │ │
│  └────────────┬─────────────────────────────────────────────────┘ │
│               ↑ INFORMS, VALIDATES, AGGREGATES                     │
└───────────────┼────────────────────────────────────────────────────┘
                │
┌───────────────┼────────────────────────────────────────────────────┐
│  LAYER 6: TRACK (Observations & Metrics)                           │
│  ┌────────────┴─────────────────────────────────────────────────┐ │
│  │  Metric (unit-based)   Observation (timestamped)             │ │
│  │  Event (occurrences)   Signal (strength-based)               │ │
│  └────────────┬─────────────────────────────────────────────────┘ │
│               ↑ TRACKS, OBSERVES                                   │
└───────────────┼────────────────────────────────────────────────────┘
                │
┌───────────────┼────────────────────────────────────────────────────┐
│  LAYER 5: EXECUTION (Daily Actions & Tasks)                        │
│  ┌────────────┴─────────────────────────────────────────────────┐ │
│  │  Decision (ROI-scored)   Task (effort-estimated)             │ │
│  │  TimeBlock (calendar)    WorkSession (logged time)           │ │
│  │  Action (atomic verbs)                                       │ │
│  └────────────┬─────────────────────────────────────────────────┘ │
│               ↑ HAS_TASK, SCHEDULED_AS, LOGGED_AS, SERVES          │
└───────────────┼────────────────────────────────────────────────────┘
                │
┌───────────────┼────────────────────────────────────────────────────┐
│  LAYER 4: TACTICS (Projects & Systems)                             │
│  ┌────────────┴─────────────────────────────────────────────────┐ │
│  │  Project (phase-based)   Area (ongoing domains)              │ │
│  │  System (dependencies)   Process (recurring cadence)         │ │
│  └────────────┬─────────────────────────────────────────────────┘ │
│               ↑ HAS_PROJECT, DEPENDS_ON, USES                      │
└───────────────┼────────────────────────────────────────────────────┘
                │
┌───────────────┼────────────────────────────────────────────────────┐
│  LAYER 3: STRATEGY (OKRs & Initiatives)                            │
│  ┌────────────┴─────────────────────────────────────────────────┐ │
│  │  Strategy (campaign)     Objective (measurable)              │ │
│  │  KeyResult (metric)      Initiative (execution vehicle)      │ │
│  │  Milestone (checkpoints) Risk (probability × impact)         │ │
│  └────────────┬─────────────────────────────────────────────────┘ │
│               ↑ SERVES, ADVANCES, CONTAINS, HAS_KEY_RESULT         │
└───────────────┼────────────────────────────────────────────────────┘
                │
┌───────────────┼────────────────────────────────────────────────────┐
│  LAYER 2: VISION (Direction & Goals)                               │
│  ┌────────────┴─────────────────────────────────────────────────┐ │
│  │  Vision (horizons)       Business (operating entities)       │ │
│  │  Goal (time-bounded)     Season (temporal windows)           │ │
│  └────────────┬─────────────────────────────────────────────────┘ │
│               ↑ ALIGNS_WITH, SERVES, ENABLES, HAS_GOAL             │
└───────────────┼────────────────────────────────────────────────────┘
                │
┌───────────────┼────────────────────────────────────────────────────┐
│  LAYER 1: FOUNDATION (Constitutional Identity)                     │
│  ┌────────────┴─────────────────────────────────────────────────┐ │
│  │  Value (core beliefs)     Principle (operating rules)        │ │
│  │  Guardrail (boundaries)   Human (actors)                     │ │
│  │  AI (agents)              Partnership (alliances)            │ │
│  │  Policy (governance)      Context (situational framing)      │ │
│  │  Core (identity pillars)  Relationship (historical context)  │ │
│  └──────────────────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────────────────┘
```

---

## Layer-by-Layer Breakdown

### Layer 1: Foundation (Constitutional Identity)

**Purpose:** WHO WE ARE — immutable values, principles, and actors

**Node Types (10):**
- `Value` — Core beliefs (e.g., "Financial Freedom", "Human-AI Symbiosis")
- `Principle` — Operating rules derived from values (e.g., "AI Principal Primacy")
- `Guardrail` — Non-negotiable boundaries with trigger/response pairs
- `Human` — Human actors with roles, email, timezone
- `AI` — AI agents with capabilities and models
- `Partnership` — Formal alliances with criticality levels
- `Policy` — Governance statements scoped to domains
- `Context` — Situational framing (time, environment, constraints)
- `Core` — Identity pillars (e.g., "Solo+AI Operating System")
- `Relationship` — Historical relationship context

**Key Relationships:**
```
Human → HOLDS_VALUE → Value
Human → FOLLOWS_PRINCIPLE → Principle
Human → RESPECTS_GUARDRAIL → Guardrail
Value → SUPPORTED_BY → Principle
Principle → ENFORCED_BY → Guardrail
Human → PARTNERS_WITH → AI | Partnership
```

**Module Access:**
- **Write:** `discuss` (full control)
- **Read:** All modules (constitutional reference)

---

### Layer 2: Vision (Direction & Goals)

**Purpose:** WHERE WE GO — long-term direction and measurable goals

**Node Types (4):**
- `Vision` — Directional statements with time horizons (1y, 3y, 5y, 10y, lifetime)
- `Business` — Operating companies (e.g., Tipz CBS, Automated Exchange)
- `Goal` — Time-bounded goals with deadlines, targets, and completion tracking
- `Season` — Temporal execution windows (e.g., "Q4 2025: CBS Strategy")

**Key Relationships:**
```
Goal → SERVES → Vision
Goal → ALIGNS_WITH → Value | Principle (Layer 1)
Season → HAS_GOAL → Goal
Business → HAS_GOAL → Goal
Business → HAS_METRIC → Metric (Layer 6)
Vision → ENABLES → Business
Partnership → ENABLES → Vision | Goal | Business
```

**Module Access:**
- **Write:** `discuss` (vision alignment)
- **Read:** All modules (strategic context)

---

### Layer 3: Strategy (How We Achieve It)

**Purpose:** HOW WE GET THERE — OKRs, initiatives, and strategic planning

**Node Types (6):**
- `Strategy` — Campaign containers for multi-quarter efforts
- `Objective` — Measurable strategic objectives
- `KeyResult` — Quantified success metrics for objectives
- `Initiative` — Strategic initiatives advancing objectives
- `Milestone` — Progress checkpoints with success criteria
- `Risk` — Identified risks with probability, impact, mitigation

**Key Relationships:**
```
Objective → SERVES → Goal (Layer 2)
Objective → HAS_KEY_RESULT → KeyResult
Initiative → ADVANCES → Objective
Initiative → HAS_MILESTONE → Milestone
Milestone → DEPENDS_ON → Milestone (sequencing)
Initiative → HAS_RISK → Risk
Strategy → CONTAINS → Objective | KeyResult | Initiative | Risk | Milestone
```

**Module Access:**
- **Write:** `plan` (strategic decomposition)
- **Read:** All modules (OKR tracking)

---

### Layer 4: Tactics (Projects & Systems)

**Purpose:** WHAT WE BUILD — concrete projects and recurring processes

**Node Types (4):**
- `Project` — Discrete projects with phases (ideation, design, build, launch, scale)
- `Area` — Ongoing areas of responsibility (e.g., Health & Performance)
- `System` — Infrastructure dependencies (e.g., ODEI, Miro, Notion)
- `Process` — Recurring processes with cadence (e.g., "MMA 3x/week")

**Key Relationships:**
```
Initiative → HAS_PROJECT → Project (Layer 3 → 4)
Area → HAS_PROJECT → Project
Area → HAS_PROCESS → Process
Project → HAS_TASK → Task (Layer 5)
Project → DEPENDS_ON → Project
Project → USES → System
Process → GENERATES_TASK → Task (recurring)
```

**Module Access:**
- **Write:** `plan` (project planning)
- **Read:** All modules (execution context)

---

### Layer 5: Execution (Daily Actions)

**Purpose:** WHAT WE DO — tasks, decisions, time blocking, work sessions

**Node Types (5):**
- `Decision` — Strategic decisions with ROI scoring
- `Task` — Actionable tasks with effort estimates, due dates, priority
- `TimeBlock` — Calendar time blocks with calendar references
- `WorkSession` — Logged work sessions with notes
- `Action` — Atomic actions with verbs and context

**Key Relationships:**
```
Project → HAS_TASK → Task (Layer 4 → 5)
Task → SCHEDULED_AS → TimeBlock
Task → LOGGED_AS → WorkSession
Decision → SERVES → Project (Layer 4)
Decision → ADVANCES → Objective | KeyResult (Layer 3)
Task → SERVES → Goal (Layer 2)
Decision → SUPPORTED_BY → Evidence (Layer 7)
```

**Module Access:**
- **Write:** `execute` (task management)
- **Read:** All modules (workload assessment)

---

### Layer 6: Track (Observations & Metrics)

**Purpose:** WHAT WE MEASURE — performance tracking and telemetry

**Node Types (4):**
- `Metric` — Quantitative metrics with units (e.g., "Daily Active Users", unit: "users")
- `Observation` — Timestamped observations of reality
- `Event` — Significant occurrences with categories
- `Signal` — Early indicators with strength ratings

**Key Relationships:**
```
Observation → TRACKS → Metric
Observation → OBSERVES → Goal | Project | Business (Layers 2, 4)
Observation → INFORMS → Insight | Pattern (Layer 7)
Business → HAS_METRIC → Metric (Layer 2 → 6)
```

**Module Access:**
- **Write:** `execute`, `mind` (telemetry)
- **Read:** All modules (metrics dashboards)

---

### Layer 7: Mind (Learning & Insights)

**Purpose:** WHAT WE LEARN — pattern recognition and institutional memory

**Node Types (5):**
- `Insight` — Discovered insights from data with confidence scores
- `Pattern` — Recurring behavioral patterns with sample sizes
- `Evidence` — Supporting evidence for insights (ventures, experiences)
- `Source` — External sources with URIs and retrieval timestamps
- `Note` — Hub nodes for aggregating institutional memory

**Key Relationships:**
```
Observation → INFORMS → Insight | Pattern (Layer 6 → 7)
Evidence → VALIDATES → Pattern | Insight
Insight → APPLIED_TO → Goal | Project (Layers 2, 4)
Insight → TRIGGERS → Decision (Layer 5)
Pattern → DEMONSTRATES → Value | Principle (Layer 1)
Note → AGGREGATES → Evidence | Pattern | Insight (memory hub)
Note → INFORMS_IDENTITY → Human (Layer 1)
```

**Module Access:**
- **Write:** `mind` (learning)
- **Read:** All modules (applied insights)

---

## Node Type Reference

### Common Properties (All Nodes)

Every node extends `BaseNodeCommon`:

```typescript
{
  id: string                    // UUID or deterministic from idempotencyKey
  title: string                 // 1-160 chars, required
  summary: string               // 1-2000 chars, REQUIRED (enforced by APOC trigger)
  description?: string          // max 4000 chars
  status: "draft" | "active" | "archived" | "deprecated"  // default: "active"
  tags?: string[]               // max 20 tags
  metadata?: Record<string, Primitive>  // PRIMITIVES ONLY (no nested objects)
}
```

**Critical Constraints:**
- `summary` field is **required** for all nodes (APOC trigger enforces this)
- `metadata` values must be primitives or arrays of primitives
- Nested objects in `metadata` will cause database errors

**Provenance (on all writes):**
```typescript
{
  module: "discuss" | "plan" | "execute" | "mind"
  actor: "human" | "agent" | "joint"
  source: string      // Tool name, conversation ID, etc.
  confidence: number  // 0.0-1.0
  notes?: string      // Optional context (max 500 chars)
}
```

---

### Foundation Layer Nodes

#### Value
```typescript
{
  ...BaseNodeCommon,
  priority?: "core" | "supporting" | "exploratory"
}
```
**Example:** "Financial Freedom", "Human-AI Symbiosis"

#### Principle
```typescript
{
  ...BaseNodeCommon,
  statement: string,    // Required: the principle statement
  rationale?: string    // Why this principle exists
}
```
**Example:** "AI Principal Primacy" — AI operates as equal principal, not assistant

#### Guardrail
```typescript
{
  ...BaseNodeCommon,
  trigger: string,      // Required: condition that activates guardrail
  response: string,     // Required: action when triggered
  severity?: "low" | "medium" | "high"
}
```
**Example:** Trigger: "Evening strategic pivot", Response: "Defer until morning 09:00-14:00"

#### Human
```typescript
{
  ...BaseNodeCommon,
  email?: string,
  role?: string,
  timezone?: string
}
```

#### AI
```typescript
{
  ...BaseNodeCommon,
  model?: string,               // e.g., "claude-opus-4.5"
  capabilities?: string[]       // e.g., ["strategic_thinking", "code_generation"]
}
```

#### Partnership
```typescript
{
  ...BaseNodeCommon,
  partner: string,              // Required: partner entity name
  criticality?: "low" | "medium" | "high",
  scope?: string
}
```

#### Core
```typescript
{
  ...BaseNodeCommon,
  pillar: string,               // Required: identity pillar (e.g., "Solo+AI")
  scope?: string
}
```

---

### Vision Layer Nodes

#### Vision
```typescript
{
  ...BaseNodeCommon,
  horizon?: "1y" | "3y" | "5y" | "10y" | "lifetime" | "custom",
  narrative?: string            // Long-form vision narrative
}
```

#### Business
```typescript
{
  ...BaseNodeCommon,
  businessModel: string,        // Required: e.g., "SaaS", "Marketplace"
  stage?: "ideation" | "mvp" | "growth" | "scale",
  revenue?: {
    model?: string,
    current?: string,
    target?: string
  },
  equity?: {
    stake?: string,
    valuation?: string
  },
  team?: {
    size?: number,
    target?: number,
    keyPeople?: string[]
  },
  markets?: string[],
  competitors?: string[]
}
```

#### Goal
```typescript
{
  ...BaseNodeCommon,
  horizon?: "week" | "month" | "quarter" | "year" | "custom",
  target?: string,
  deadline?: string,            // ISO datetime
  plannedDeadline?: string,     // Original plan
  actualDeadline?: string,      // Actual completion
  delayMonths?: number,
  completionStatus?: "successfully_completed" | "partially_completed" |
                     "abandoned" | "superseded",
  completedAt?: string,
  deliverables?: string,
  proofSource?: string
}
```

#### Season
```typescript
{
  ...BaseNodeCommon,
  startDate: string,            // Required: ISO date
  endDate: string,              // Required: ISO date
  theme?: string
}
```

---

### Strategy Layer Nodes

#### Strategy
```typescript
{
  ...BaseNodeCommon,
  narrative?: string,
  timebox?: string              // Duration (e.g., "Q4 2025 - Q1 2026")
}
```

#### Objective
```typescript
{
  ...BaseNodeCommon,
  timeframe: string,            // Required: when objective should be achieved
  successCriteria?: string[]
}
```

#### KeyResult
```typescript
{
  ...BaseNodeCommon,
  metric: string,               // Required: what we're measuring
  target: number | string,      // Required: target value
  baseline?: number | string
}
```

#### Initiative
```typescript
{
  ...BaseNodeCommon,
  impact?: "low" | "medium" | "high",
  owner?: string
}
```

#### Milestone
```typescript
{
  ...BaseNodeCommon,
  successCriteria: string,      // Required: how we know it's done
  deadline?: string,
  owner?: string,
  completedAt?: string,
  successEvidence?: string,
  blockedReason?: string
}
```

#### Risk
```typescript
{
  ...BaseNodeCommon,
  probability: number,          // Required: 0.0-1.0
  impact: "low" | "medium" | "high",  // Required
  mitigation?: string
}
```

---

### Tactics Layer Nodes

#### Project
```typescript
{
  ...BaseNodeCommon,
  phase?: "ideation" | "design" | "build" | "launch" | "scale",
  owner?: string
}
```

#### Area
```typescript
{
  ...BaseNodeCommon,
  domain?: string               // e.g., "Health & Performance"
}
```

#### System
```typescript
{
  ...BaseNodeCommon,
  scope?: string,
  owner?: string
}
```

#### Process
```typescript
{
  ...BaseNodeCommon,
  cadence?: string,             // e.g., "3x/week", "daily"
  owner?: string
}
```

---

### Execution Layer Nodes

#### Decision
```typescript
{
  ...BaseNodeCommon,
  decidedAt?: string,           // ISO datetime
  roi?: {
    financial?: number,
    time?: number,
    alignment?: number,
    risk?: number,
    total?: number
  }
}
```

#### Task
```typescript
{
  ...BaseNodeCommon,
  status: "todo" | "in_progress" | "blocked" | "done",  // Required (default: "todo")
  priority?: "p0" | "p1" | "p2" | "p3",
  effortHours?: number,
  dueDate?: string,
  owner?: string
}
```

#### TimeBlock
```typescript
{
  ...BaseNodeCommon,
  start: string,                // Required: ISO datetime
  end: string,                  // Required: ISO datetime
  calendarRef?: {
    provider: "apple",
    externalId: string
  }
}
```

#### WorkSession
```typescript
{
  ...BaseNodeCommon,
  start: string,                // Required: ISO datetime
  end: string,                  // Required: ISO datetime
  notes?: string
}
```

#### Action
```typescript
{
  ...BaseNodeCommon,
  verb: string,                 // Required: action verb (e.g., "ship", "validate")
  context?: string
}
```

---

### Track Layer Nodes

#### Metric
```typescript
{
  ...BaseNodeCommon,
  unit: string,                 // Required: measurement unit (e.g., "users", "$", "hours")
  direction?: "up" | "down" | "stable",
  cadence?: string              // How often measured (e.g., "daily", "weekly")
}
```

#### Observation
```typescript
{
  ...BaseNodeCommon,
  observedAt: string,           // Required: ISO datetime
  source?: string
}
```

#### Event
```typescript
{
  ...BaseNodeCommon,
  occurredAt: string,           // Required: ISO datetime
  category?: string
}
```

#### Signal
```typescript
{
  ...BaseNodeCommon,
  strength?: "low" | "medium" | "high",
  channel?: string
}
```

---

### Mind Layer Nodes

#### Insight
```typescript
{
  ...BaseNodeCommon,
  confidence: number,           // Required: 0.0-1.0
  insightType?: string,
  sourceRef?: string
}
```

#### Pattern
```typescript
{
  ...BaseNodeCommon,
  pattern: string,              // Required: pattern description
  confidence?: number           // 0.0-1.0
}
```

#### Evidence
```typescript
{
  ...BaseNodeCommon,
  evidenceType?: "quantitative" | "qualitative" | "anecdotal",
  sourceRef?: string
}
```

#### Source
```typescript
{
  ...BaseNodeCommon,
  uri: string,                  // Required: source URI/URL
  retrievedAt?: string
}
```

#### Note
```typescript
{
  ...BaseNodeCommon,
  content: string               // Required: note body
}
```

---

## Relationship Flow Patterns

### Constitutional Alignment (Foundation → Vision)

```
Value/Principle → ALIGNS_WITH → Vision/Goal
```
Ensures all goals align with constitutional values.

### Strategic Decomposition (Vision → Strategy → Tactics → Execution)

```
Vision → SERVES (Goal)
Goal → SERVES (Objective)
Objective → HAS_KEY_RESULT (KeyResult)
Objective → ADVANCES (Initiative)
Initiative → HAS_PROJECT (Project)
Project → HAS_TASK (Task)
```

### Observation → Learning (Track → Mind)

```
Observation → TRACKS (Metric)
Observation → OBSERVES (Goal/Project/Business)
Observation → INFORMS (Insight/Pattern)
```

### Applied Learning (Mind → Execution)

```
Insight → APPLIED_TO (Goal/Project)
Insight → TRIGGERS (Decision)
Evidence → VALIDATES (Pattern/Insight)
Pattern → DEMONSTRATES (Value/Principle)
```

### Time & Capacity Management

```
Task → SCHEDULED_AS (TimeBlock)
Task → LOGGED_AS (WorkSession)
Process → GENERATES_TASK (Task)
```

---

## Cross-Layer Connections

### Upward Traceability (Execution → Foundation)

Every execution element can trace back to constitutional values:

```
Task → SERVES → Goal → ALIGNS_WITH → Value
Decision → ADVANCES → Objective → SERVES → Goal → ALIGNS_WITH → Principle
```

### Downward Enablement (Foundation → Execution)

Foundation elements enable execution:

```
Partnership → ENABLES → Business → HAS_GOAL → Goal → HAS_PROJECT → Project → HAS_TASK → Task
```

### Feedback Loops (Execution → Track → Mind → Foundation)

Learning informs future execution:

```
Task completion → Observation → INFORMS → Pattern → DEMONSTRATES → Principle
Pattern detection → Insight → APPLIED_TO → Goal (future planning)
```

---

**Next:** See [QUERY-COOKBOOK.md](./QUERY-COOKBOOK.md) for Cypher query patterns and [PROVENANCE.md](./PROVENANCE.md) for provenance tracking details.
