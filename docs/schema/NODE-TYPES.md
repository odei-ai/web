# ODEI Node Types — Quick Reference

**Version:** 1.0
**Generated:** 2025-12-25
**Total Node Types:** 38

---

## Table of Contents

1. [Foundation Layer (10 types)](#foundation-layer-10-types)
2. [Vision Layer (4 types)](#vision-layer-4-types)
3. [Strategy Layer (6 types)](#strategy-layer-6-types)
4. [Tactics Layer (4 types)](#tactics-layer-4-types)
5. [Execution Layer (5 types)](#execution-layer-5-types)
6. [Track Layer (4 types)](#track-layer-4-types)
7. [Mind Layer (5 types)](#mind-layer-5-types)
8. [Common Properties](#common-properties)
9. [Field Type Reference](#field-type-reference)

---

## Foundation Layer (10 types)

### Value
**Purpose:** Core beliefs and principles
**Required:** `title`, `summary`, `description`
**Optional:** `priority`
**Priority:** `"core"` | `"supporting"` | `"exploratory"`

**Example:**
```json
{
  "title": "Financial Freedom",
  "summary": "Build wealth to enable complete autonomy and generational impact",
  "description": "Financial freedom means...",
  "priority": "core"
}
```

---

### Principle
**Purpose:** Operating rules derived from values
**Required:** `title`, `summary`, `statement`
**Optional:** `rationale`

**Example:**
```json
{
  "title": "AI Principal Primacy",
  "summary": "AI operates as equal principal, not assistant",
  "statement": "In ODEI Symbiosis, Claude is AI Principal with strategic authority",
  "rationale": "Enables true human-AI partnership..."
}
```

---

### Guardrail
**Purpose:** Non-negotiable boundaries with triggers
**Required:** `title`, `summary`, `trigger`, `response`
**Optional:** `severity`
**Severity:** `"low"` | `"medium"` | `"high"`

**Example:**
```json
{
  "title": "No Evening Strategic Pivots",
  "summary": "Strategic decisions only during morning hours",
  "trigger": "Strategic decision attempt after 20:00",
  "response": "Defer to morning 09:00-14:00 window",
  "severity": "high"
}
```

---

### Human
**Purpose:** Human actors in the system
**Required:** `title`, `summary`
**Optional:** `email`, `role`, `timezone`

**Example:**
```json
{
  "title": "Anton Illarionov",
  "summary": "Human Principal in ODEI Symbiosis",
  "email": "anton@example.com",
  "role": "Founder",
  "timezone": "Asia/Tbilisi"
}
```

---

### AI
**Purpose:** AI agents in the system
**Required:** `title`, `summary`
**Optional:** `model`, `capabilities`

**Example:**
```json
{
  "title": "Claude (AI Principal)",
  "summary": "AI Principal in ODEI Symbiosis",
  "model": "claude-opus-4.5",
  "capabilities": ["strategic_thinking", "pattern_recognition", "code_generation"]
}
```

---

### Partnership
**Purpose:** Formal alliances and partnerships
**Required:** `title`, `summary`, `partner`
**Optional:** `criticality`, `scope`
**Criticality:** `"low"` | `"medium"` | `"high"`

**Example:**
```json
{
  "title": "Tipz.ae Co-Founder Partnership",
  "summary": "7% equity CPO stake in Tipz CBS",
  "partner": "Tipz.ae Founders",
  "criticality": "high",
  "scope": "Product strategy, UX/UI, marketplace dynamics"
}
```

---

### Policy
**Purpose:** Governance statements
**Required:** `title`, `summary`, `statement`
**Optional:** `scope`

**Example:**
```json
{
  "title": "AI Principal Decision Authority",
  "summary": "AI Principal has authority over strategic recommendations",
  "statement": "Claude may recommend strategic pivots without human pre-approval",
  "scope": "strategic_planning"
}
```

---

### Context
**Purpose:** Situational framing
**Required:** `title`, `summary`, `scope`
**Optional:** `timeframe`

**Example:**
```json
{
  "title": "Parenthood Context",
  "summary": "Active parent with young children",
  "scope": "family_constraints",
  "timeframe": "2024-2040"
}
```

---

### Core
**Purpose:** Identity pillars
**Required:** `title`, `summary`, `pillar`
**Optional:** `scope`

**Example:**
```json
{
  "title": "Solo+AI Operating System",
  "summary": "Solo founder + AI co-principal model",
  "pillar": "operational_identity",
  "scope": "business_building"
}
```

---

### Relationship
**Purpose:** Historical relationship context
**Required:** `title`, `summary`, `counterpart`
**Optional:** `nature`, `since`

**Example:**
```json
{
  "title": "Nisanov Family History",
  "summary": "Long-term relationship context",
  "counterpart": "Nisanov Family Office",
  "nature": "professional_historical",
  "since": "2015"
}
```

---

## Vision Layer (4 types)

### Vision
**Purpose:** Directional statements across time horizons
**Required:** `title`, `summary`
**Optional:** `horizon`, `narrative`
**Horizon:** `"1y"` | `"3y"` | `"5y"` | `"10y"` | `"lifetime"` | `"custom"`

**Example:**
```json
{
  "title": "Generational Financial Freedom",
  "summary": "Build $500M net worth enabling multi-generational impact",
  "horizon": "10y",
  "narrative": "Long-form vision narrative..."
}
```

---

### Business
**Purpose:** Operating companies
**Required:** `title`, `summary`, `businessModel`
**Optional:** `stage`, `revenue`, `equity`, `team`, `markets`, `competitors`
**Stage:** `"ideation"` | `"mvp"` | `"growth"` | `"scale"`

**Example:**
```json
{
  "title": "Tipz CBS",
  "summary": "Sovereign banking marketplace",
  "businessModel": "Marketplace SaaS",
  "stage": "mvp",
  "revenue": {
    "model": "commission + subscription",
    "target": "$500K MRR"
  },
  "equity": {
    "stake": "7%",
    "valuation": "TBD"
  },
  "markets": ["UAE", "Georgia", "Global"]
}
```

---

### Goal
**Purpose:** Time-bounded goals
**Required:** `title`, `summary`
**Optional:** `horizon`, `target`, `deadline`, `completionStatus`, `deliverables`
**Horizon:** `"week"` | `"month"` | `"quarter"` | `"year"` | `"custom"`
**CompletionStatus:** `"successfully_completed"` | `"partially_completed"` | `"abandoned"` | `"superseded"`

**Example:**
```json
{
  "title": "$500M Net Worth by 2035",
  "summary": "Build $500M net worth through portfolio of businesses",
  "horizon": "10y",
  "target": "$500,000,000",
  "deadline": "2035-01-01",
  "deliverables": "Liquid net worth of $500M+"
}
```

---

### Season
**Purpose:** Temporal execution windows
**Required:** `title`, `summary`, `startDate`, `endDate`
**Optional:** `theme`

**Example:**
```json
{
  "title": "Q4 2025: CBS Strategy",
  "summary": "Complete CBS strategy package and pitch",
  "startDate": "2025-10-01",
  "endDate": "2025-12-31",
  "theme": "Strategic Planning & Partnership"
}
```

---

## Strategy Layer (6 types)

### Strategy
**Purpose:** Campaign containers for multi-quarter efforts
**Required:** `title`, `summary`
**Optional:** `narrative`, `timebox`

**Example:**
```json
{
  "title": "CBS Q4 2025 - Q1 2026",
  "summary": "Complete strategy package and launch MVP",
  "narrative": "Multi-quarter strategy narrative...",
  "timebox": "2025-Q4 through 2026-Q1"
}
```

---

### Objective
**Purpose:** Measurable strategic objectives
**Required:** `title`, `summary`, `timeframe`
**Optional:** `successCriteria`

**Example:**
```json
{
  "title": "Build $500M Equity Portfolio",
  "summary": "Construct portfolio of businesses totaling $500M equity value",
  "timeframe": "2025-2035",
  "successCriteria": ["$500M+ liquid net worth", "3+ successful exits"]
}
```

---

### KeyResult
**Purpose:** Quantified success metrics for objectives
**Required:** `title`, `summary`, `metric`, `target`
**Optional:** `baseline`

**Example:**
```json
{
  "title": "Reach 100 Paying Customers",
  "summary": "Grow to 100 paying customers by Q2 2026",
  "metric": "paying_customers",
  "target": 100,
  "baseline": 0
}
```

---

### Initiative
**Purpose:** Strategic initiatives advancing objectives
**Required:** `title`, `summary`
**Optional:** `impact`, `owner`
**Impact:** `"low"` | `"medium"` | `"high"`

**Example:**
```json
{
  "title": "Tipz Partnership Initiative",
  "summary": "Execute CPO role to build Tipz CBS MVP",
  "impact": "high",
  "owner": "Anton Illarionov"
}
```

---

### Milestone
**Purpose:** Progress checkpoints with success criteria
**Required:** `title`, `summary`, `successCriteria`
**Optional:** `deadline`, `owner`, `completedAt`, `successEvidence`, `blockedReason`

**Example:**
```json
{
  "title": "Dec 6 Strategy Complete",
  "summary": "Complete CBS strategy package by Dec 6",
  "successCriteria": "Strategy doc finalized and approved",
  "deadline": "2025-12-06",
  "owner": "Anton Illarionov"
}
```

---

### Risk
**Purpose:** Identified risks with probability and impact
**Required:** `title`, `summary`, `probability`, `impact`
**Optional:** `mitigation`
**Impact:** `"low"` | `"medium"` | `"high"`

**Example:**
```json
{
  "title": "Partnership Execution Risk",
  "summary": "Risk that partnership doesn't execute as planned",
  "probability": 0.35,
  "impact": "high",
  "mitigation": "Regular check-ins, clear success criteria"
}
```

---

## Tactics Layer (4 types)

### Project
**Purpose:** Discrete projects with deliverables
**Required:** `title`, `summary`
**Optional:** `phase`, `owner`
**Phase:** `"ideation"` | `"design"` | `"build"` | `"launch"` | `"scale"`

**Example:**
```json
{
  "title": "Tipz Integration API",
  "summary": "Build API integration for Tipz CBS marketplace",
  "phase": "build",
  "owner": "Anton Illarionov"
}
```

---

### Area
**Purpose:** Ongoing areas of responsibility
**Required:** `title`, `summary`
**Optional:** `domain`

**Example:**
```json
{
  "title": "Health & Performance",
  "summary": "Physical fitness, MMA training, nutrition",
  "domain": "personal_development"
}
```

---

### System
**Purpose:** Infrastructure dependencies
**Required:** `title`, `summary`
**Optional:** `scope`, `owner`

**Example:**
```json
{
  "title": "ODEI Memory Atlas",
  "summary": "Neo4j constitutional graph database",
  "scope": "institutional_memory",
  "owner": "ODEI Infrastructure"
}
```

---

### Process
**Purpose:** Recurring processes with cadence
**Required:** `title`, `summary`
**Optional:** `cadence`, `owner`

**Example:**
```json
{
  "title": "MMA Training 3x/week",
  "summary": "Mixed martial arts training sessions",
  "cadence": "3x/week",
  "owner": "Anton Illarionov"
}
```

---

## Execution Layer (5 types)

### Decision
**Purpose:** Strategic decisions with ROI scoring
**Required:** `title`, `summary`
**Optional:** `decidedAt`, `roi`

**Example:**
```json
{
  "title": "Hire First Engineer",
  "summary": "Decision to hire first full-time engineer",
  "decidedAt": "2025-12-20T10:00:00Z",
  "roi": {
    "financial": 0.8,
    "time": 0.9,
    "alignment": 1.0,
    "risk": 0.7,
    "total": 0.85
  }
}
```

---

### Task
**Purpose:** Actionable tasks with effort estimates
**Required:** `title`, `summary`
**Optional:** `status`, `priority`, `effortHours`, `dueDate`, `owner`
**Status:** `"todo"` | `"in_progress"` | `"blocked"` | `"done"`
**Priority:** `"p0"` | `"p1"` | `"p2"` | `"p3"`

**Example:**
```json
{
  "title": "Implement Auth Endpoint",
  "summary": "Build authentication endpoint for Tipz API",
  "status": "in_progress",
  "priority": "p1",
  "effortHours": 8,
  "dueDate": "2025-12-31",
  "owner": "Anton Illarionov"
}
```

---

### TimeBlock
**Purpose:** Calendar time blocks
**Required:** `title`, `summary`, `start`, `end`
**Optional:** `calendarRef`

**Example:**
```json
{
  "title": "Deep Work: Auth Implementation",
  "summary": "Focus block for authentication work",
  "start": "2025-12-25T09:00:00Z",
  "end": "2025-12-25T13:00:00Z",
  "calendarRef": {
    "provider": "apple",
    "externalId": "cal_event_123"
  }
}
```

---

### WorkSession
**Purpose:** Logged work sessions
**Required:** `title`, `summary`, `start`, `end`
**Optional:** `notes`

**Example:**
```json
{
  "title": "Auth Implementation Session",
  "summary": "Completed auth endpoint implementation",
  "start": "2025-12-25T09:00:00Z",
  "end": "2025-12-25T13:00:00Z",
  "notes": "Finished JWT implementation, added tests"
}
```

---

### Action
**Purpose:** Atomic actions with verbs
**Required:** `title`, `summary`, `verb`
**Optional:** `context`

**Example:**
```json
{
  "title": "Ship: Auth Endpoint",
  "summary": "Deploy auth endpoint to production",
  "verb": "ship",
  "context": "Production deployment after QA approval"
}
```

---

## Track Layer (4 types)

### Metric
**Purpose:** Quantitative metrics with units
**Required:** `title`, `summary`, `unit`
**Optional:** `direction`, `cadence`
**Direction:** `"up"` | `"down"` | `"stable"`

**Example:**
```json
{
  "title": "Daily Active Users",
  "summary": "Number of daily active users",
  "unit": "users",
  "direction": "up",
  "cadence": "daily"
}
```

---

### Observation
**Purpose:** Timestamped observations
**Required:** `title`, `summary`, `observedAt`
**Optional:** `source`

**Example:**
```json
{
  "title": "MRR Growth Feb 2024",
  "summary": "MRR increased to $50K in February",
  "observedAt": "2024-02-28T12:00:00Z",
  "source": "financial_dashboard"
}
```

---

### Event
**Purpose:** Significant occurrences
**Required:** `title`, `summary`, `occurredAt`
**Optional:** `category`

**Example:**
```json
{
  "title": "Aegis.im Exit",
  "summary": "Successful exit from Aegis.im marketplace",
  "occurredAt": "2024-03-15T12:00:00Z",
  "category": "business_milestone"
}
```

---

### Signal
**Purpose:** Early indicators
**Required:** `title`, `summary`
**Optional:** `strength`, `channel`
**Strength:** `"low"` | `"medium"` | `"high"`

**Example:**
```json
{
  "title": "User Engagement Spike",
  "summary": "Unusual increase in user engagement detected",
  "strength": "medium",
  "channel": "analytics_dashboard"
}
```

---

## Mind Layer (5 types)

### Insight
**Purpose:** Discovered insights with confidence scores
**Required:** `title`, `summary`, `confidence`
**Optional:** `insightType`, `sourceRef`

**Example:**
```json
{
  "title": "Partnership Revenue Driver",
  "summary": "Partnerships are the primary revenue driver",
  "confidence": 0.87,
  "insightType": "revenue_analysis",
  "sourceRef": "mrr_growth_feb_2024"
}
```

---

### Pattern
**Purpose:** Recurring behavioral patterns
**Required:** `title`, `summary`, `pattern`
**Optional:** `confidence`

**Example:**
```json
{
  "title": "Solo+AI Delivers",
  "summary": "Solo founder + AI co-principal yields successful exits",
  "pattern": "Solo + AI partnership model → Successful venture outcomes",
  "confidence": 0.9
}
```

---

### Evidence
**Purpose:** Supporting evidence
**Required:** `title`, `summary`
**Optional:** `evidenceType`, `sourceRef`
**EvidenceType:** `"quantitative"` | `"qualitative"` | `"anecdotal"`

**Example:**
```json
{
  "title": "Aegis.im Exit",
  "summary": "Successful marketplace exit in 2024",
  "evidenceType": "quantitative",
  "sourceRef": "financial_records_2024"
}
```

---

### Source
**Purpose:** External sources
**Required:** `title`, `summary`, `uri`
**Optional:** `retrievedAt`

**Example:**
```json
{
  "title": "Neo4j Graph Database Documentation",
  "summary": "Official Neo4j documentation",
  "uri": "https://neo4j.com/docs/",
  "retrievedAt": "2025-12-25T09:00:00Z"
}
```

---

### Note
**Purpose:** Hub nodes for institutional memory
**Required:** `title`, `summary`, `content`

**Example:**
```json
{
  "title": "Institutional Memory",
  "summary": "Central hub for career history and patterns",
  "content": "Aggregated learnings from ventures, exits, and experiences"
}
```

---

## Common Properties

All nodes include these base properties:

```typescript
{
  id: string,                    // UUID or deterministic from idempotencyKey
  title: string,                 // 1-160 chars, required
  summary: string,               // 1-2000 chars, REQUIRED (APOC enforced)
  description?: string,          // max 4000 chars
  status: "draft" | "active" | "archived" | "deprecated",  // default: "active"
  tags?: string[],               // max 20 tags
  metadata?: Record<string, Primitive>,  // PRIMITIVES ONLY
  createdAt: string,             // ISO datetime (auto-generated)
  updatedAt: string,             // ISO datetime (auto-updated)
  provenance: {                  // Required on all writes
    module: string,
    actor: string,
    source: string,
    confidence: number,
    notes?: string
  }
}
```

---

## Field Type Reference

### String Fields
- `title` — 1-160 characters
- `summary` — 1-2000 characters (REQUIRED)
- `description` — max 4000 characters
- `notes` — max 500 characters (in provenance)

### Enum Fields
- `status` — `"draft"` | `"active"` | `"archived"` | `"deprecated"`
- `priority` (Value) — `"core"` | `"supporting"` | `"exploratory"`
- `severity` (Guardrail) — `"low"` | `"medium"` | `"high"`
- `horizon` (Vision) — `"1y"` | `"3y"` | `"5y"` | `"10y"` | `"lifetime"` | `"custom"`
- `horizon` (Goal) — `"week"` | `"month"` | `"quarter"` | `"year"` | `"custom"`
- `impact` (Initiative, Risk) — `"low"` | `"medium"` | `"high"`
- `phase` (Project) — `"ideation"` | `"design"` | `"build"` | `"launch"` | `"scale"`
- `status` (Task) — `"todo"` | `"in_progress"` | `"blocked"` | `"done"`
- `priority` (Task) — `"p0"` | `"p1"` | `"p2"` | `"p3"`
- `direction` (Metric) — `"up"` | `"down"` | `"stable"`
- `strength` (Signal) — `"low"` | `"medium"` | `"high"`
- `evidenceType` — `"quantitative"` | `"qualitative"` | `"anecdotal"`

### Numeric Fields
- `probability` (Risk) — 0.0-1.0
- `confidence` (Insight) — 0.0-1.0
- `effortHours` (Task) — >= 0
- `target` (KeyResult) — number or string
- `baseline` (KeyResult) — number or string

### DateTime Fields
All ISO 8601 format: `"2025-12-25T09:00:00Z"`
- `deadline`, `plannedDeadline`, `actualDeadline` (Goal)
- `decidedAt` (Decision)
- `dueDate` (Task)
- `start`, `end` (TimeBlock, WorkSession)
- `observedAt` (Observation)
- `occurredAt` (Event)
- `retrievedAt` (Source)
- `completedAt` (Goal, Milestone)

### Complex Objects
- `roi` (Decision) — `{ financial, time, alignment, risk, total }`
- `revenue` (Business) — `{ model, current, target }`
- `equity` (Business) — `{ stake, valuation }`
- `team` (Business) — `{ size, target, keyPeople }`
- `calendarRef` (TimeBlock) — `{ provider, externalId }`

### Array Fields
- `tags` — string[] (max 20)
- `capabilities` (AI) — string[]
- `successCriteria` (Objective, Milestone) — string[]
- `markets`, `competitors` (Business) — string[]
- `keyPeople` (Business team) — string[]

### Metadata Constraints
**CRITICAL:** Only primitives allowed (no nested objects)

✅ **ALLOWED:**
```json
{
  "metadata": {
    "importSource": "notion_2025_12_25",
    "externalId": "abc123",
    "priority": 1,
    "verified": true,
    "tags": ["important", "urgent"]
  }
}
```

❌ **FORBIDDEN:**
```json
{
  "metadata": {
    "config": {
      "nested": "object"  // ❌ Neo4j will reject
    }
  }
}
```

**Workaround for complex data:**
```json
{
  "metadata": {
    "config": "{\"nested\":\"object\"}"  // ✅ Serialize as JSON string
  }
}
```

---

**Next:** See [GRAPH-MODEL.md](./GRAPH-MODEL.md) for visual architecture and [QUERY-COOKBOOK.md](./QUERY-COOKBOOK.md) for usage patterns.
