# World Model Filters Audit — ODEI v2.5.4

**Date:** 2026-01-29
**Purpose:** Document all current filters for architecture alignment review
**Target:** GPT-5.2-Pro analysis

---

## Executive Summary

ODEI World Model has **multi-layered filtering** across UI, graph renderer, and backend. The v2.5.4 architecture introduces **6 Domains** and **5 Loop Phases** which are NOT yet reflected in UI filters.

**Current State:**
- UI filters: 7 Layers + 47 Node Types + Relationship types + Search
- Backend filters: Layers + Status + Memory stages
- NEW v2.5.4: 6 Domains + 5 Loop Phases + 10 Guard Queries (not in UI)

**Gap:** UI filters are layer-based (old ontology), but v2.5.4 architecture is domain-based + loop-phase-based.

---

## 1. CURRENT UI FILTERS (Atlas Panel)

### 1.1 Layer Filters (7 Layers)
| Layer | Node Count Display | Checkbox |
|-------|-------------------|----------|
| Foundation | `[data-layer-count="foundation"]` | `input[data-layer="foundation"]` |
| Vision | `[data-layer-count="vision"]` | `input[data-layer="vision"]` |
| Strategy | `[data-layer-count="strategy"]` | `input[data-layer="strategy"]` |
| Tactics | `[data-layer-count="tactics"]` | `input[data-layer="tactics"]` |
| Execution | `[data-layer-count="execution"]` | `input[data-layer="execution"]` |
| Track | `[data-layer-count="track"]` | `input[data-layer="track"]` |
| Mind | `[data-layer-count="mind"]` | `input[data-layer="mind"]` |

### 1.2 Node Type Filters (47 Types)

**Foundation (13):** Core, Value, Principle, Policy, Guardrail, AI, Human, Skill, Partnership, Resource, Context, Constraint, Relationship

**Vision (4):** Vision, Goal, Season, Business

**Strategy (5):** Strategy, Objective, Key Result, Initiative, Risk

**Tactics (6):** Project, Area, System, Capability, Process, Opportunity

**Execution (7):** Decision, Task, Time Block, Work Session, Action, Habit, Routine

**Track (7):** Metric, Observation, Event, Signal, Measurement, Meeting, Interaction

**Mind (8):** Insight, Pattern, Evidence, Source, Note, Lesson, Document, Resource

### 1.3 Relationship Filter
- Dynamic pill buttons for each relationship type
- Single-select (clicking active clears filter)
- ~50 relationship types available

### 1.4 Search
- Input: `#memory-search-input`
- Searches: title, summary, description, type, layer
- Debounced 300ms

### 1.5 Other Controls
| Control | Element | Function |
|---------|---------|----------|
| ROI Range | `#memory-roi-min/max` | Filter by ROI value |
| Date Range | `#memory-date-from/to` | Filter by date |
| Time Window | `[data-time-window]` | Last 7/30/90 days |
| Complexity | `#memory-complexity-slider` | 0-100, affects visibility budget |
| Show Edges | `#memory-toggle-edges` | Toggle edge rendering |

---

## 2. v2.5.4 ARCHITECTURE (NEW)

### 2.1 Domains (6) — NOT IN UI
```
STATE       → Foundation state, capabilities, resources
DESTINATION → Goals, vision, aspirations
PATH        → Strategy, tactics, planning
REALITY     → Execution, observations, facts
POLICY      → Rules, proposals, governance
AUDIT       → Evidence trail, logs (was: EVIDENCE)
```

**Domain Colors:**
- STATE: #4FD1C5 (Jade)
- DESTINATION: #22d3ee (Cyan)
- PATH: #5b74ff (Blue)
- REALITY: #84cc16 (Green)
- POLICY: #f59e0b (Amber)
- AUDIT: #a855f7 (Purple)

### 2.2 Loop Phases (5) — NOT IN UI
```
OBSERVE → Information gathering (Cyan)
DECIDE  → Decision making (Amber)
ACT     → Execution (Green)
VERIFY  → Verification (Purple)
EVOLVE  → Learning/adaptation (Jade)
```

### 2.3 Guard Queries (10) — Integrity Filtering
| Guard | Kind | Severity | Description |
|-------|------|----------|-------------|
| A | ORPHAN_TASK | HIGH | Task without DESTINATION link |
| B | INTENT_MISSING_TASK | HIGH | TimeBlockIntent without taskId |
| C | PROPOSAL_INVALID_OPS | MEDIUM | Proposal without exactly 1 operation |
| D | ACTION_NO_COMPLETES | HIGH | Action without COMPLETES relationship |
| E | WORKSESSION_MISSING_BOUNDS | HIGH | WorkSession without startAt/endAt |
| F | ENTITY_MISSING_ID | CRITICAL | Node without id property |
| G | ENTITY_TYPE_MISMATCH | HIGH | Domain/type schema mismatch |
| H | DECISION_NO_OBSERVATION_BASIS | HIGH | Decision without observation basis |
| I | PROPOSAL_NO_SOURCE_LINEAGE | MEDIUM | Proposal without source |
| J | GRAPHOP_UNAUDITED | CRITICAL | GraphOp without EvidenceLogEntry |

### 2.4 Pipeline Stages (5) — Partial in UI
```
Task → TimeBlockIntent → Proposal → CalendarEvent → Outcome
```

### 2.5 Status Categories
- **Terminal:** DONE, COMPLETED, CANCELLED, ARCHIVED, APPLIED, RESOLVED
- **Active:** ACTIVE, IN_PROGRESS, PENDING, NOW, COMMITTED, READY
- **Blocked:** BLOCKED, PAUSED, WAITING, ON_HOLD

---

## 3. BACKEND FILTER PARAMETERS

### 3.1 hybridSearch.ts
```typescript
{
  query: string,           // Required
  layers?: string[],       // foundation, vision, strategy, etc.
  topK?: number,           // 1-100, default: 10
  minScore?: number,       // 0-1, default: 0.3
  weights?: {
    semantic: number,      // default: 0.6
    structural: number,    // default: 0.4
  },
  expandHops?: number,     // 0-2
  excludeFields?: string[] // properties, metadata, provenance, tags
}
```

### 3.2 hybridPlusPlus.ts (Extended)
```typescript
{
  ...hybridSearch,
  weights?: {
    semantic: number,        // default: 0.35
    structural: number,      // default: 0.25
    temporal: number,        // default: 0.25 (NEW)
    personalization: number, // default: 0.15 (NEW)
  },
  enableTemporal?: boolean,
  enablePersonalization?: boolean,
  userId?: string,
  timezone?: string,
}
```

### 3.3 memoryRetrieve.ts
```typescript
{
  agent?: 'discuss' | 'plan' | 'execute' | 'mind',
  stages?: [
    'identity',       // Foundation: values, principles, guardrails
    'temporal',       // Time context: health, deadlines
    'strategic',      // Vision, goals, business
    'operational',    // Workload, tasks, capacity
    'analytics',      // Meta-memory, coverage
    'episodic',       // Last 30 days: decisions, events
    'conversational', // Last 7 days: insights, patterns
    'procedural',     // Active processes
  ],
  tokenBudget?: number,
  mode?: 'summary' | 'full',
}
```

---

## 4. GRAPH RENDERER FILTERING

### 4.1 Visibility Pipeline (10 stages)
1. Layer filter → Exclude disabled layers
2. Domain filter → Exclude disabled domains
3. Legend filter → Filter by specific type
4. Focus type → Isolate type + neighbors
5. Whitelist → Include only specified IDs
6. ROI/search/date → Value-based filters
7. Forced nodes → Add selected/pinned/focus
8. Journey corridor → Add path + halo nodes
9. Budget fill → Top-salience up to budget
10. Edge/Label policy → Final visibility

### 4.2 Visibility Budgets
```javascript
baseBudgets = {
  focus: 500,      // Focus on 1 node
  overview: 1000,  // Full graph view
  journey: 200,    // Path-finding view
}
```

### 4.3 Domain Z-Offsets (Semantic Depth)
```javascript
{
  destination: +60,  // Aspirational, "beacon in sky"
  path: +20,         // Strategic route
  state: 0,          // Current position
  reality: -10,      // Observable context
  policy: -40,       // Constraints
  evidence: -80,     // Historical trace
}
```

---

## 5. GAPS & RECOMMENDATIONS

### 5.1 Missing UI Filters

| Filter | Status | Priority |
|--------|--------|----------|
| Domain toggles (6) | Missing | HIGH |
| Loop Phase filter (5) | Missing | HIGH |
| Integrity filter (violations) | Missing | MEDIUM |
| Status filter (active/done/blocked) | Missing | MEDIUM |
| Pipeline stage filter | Partial (Board view) | LOW |
| Semantic label filter (Intent/Fact) | Missing | LOW |

### 5.2 Architecture Misalignment

**Current:** Layers (7) are primary filter dimension
**v2.5.4:** Domains (6) + Loop Phases (5) should be primary

**Proposed Filter Hierarchy:**
```
1. Domain (semantic context) - WHERE in life map
2. Loop Phase (workflow state) - WHAT stage of decision
3. Layer (constitutional depth) - HOW deep in stack
4. Type (entity kind) - WHICH specific entity
5. Status (lifecycle) - IS IT active/done/blocked
```

### 5.3 UI Filter Panel Redesign

**Current Atlas Panel:**
- Search
- Layers (accordion with types)
- Relationships
- Backups

**Proposed Atlas Panel v2.5.4:**
```
- Search
- Domains (6 toggles with counts)
- Loop Phases (5 toggles, only in Loop view)
- Layers (9, collapsed by default)
- Integrity (violations badge, quick filter)
- Relationships
- Advanced (ROI, date, time window)
```

---

## 6. TYPE-TO-DOMAIN MAPPING

### STATE Domain (17 types)
Human, Core, AI, Identity, Capability, Resource, Constraint, HealthSnapshot, Position, Context, Metric, Signal, Value, Principle, Guardrail, Asset, Policy, Partnership, Relationship

### DESTINATION Domain (8 types)
Vision, Business, Goal, Season, Value, NorthStar, KeyResult, Area

### PATH Domain (14 types)
Strategy, Objective, Initiative, Project, Path, Task, Milestone, Decision, Risk, TimeBlockIntent, System, Process, Program, Roadmap, Opportunity

### REALITY Domain (15 types)
Person, Organization, CalendarEvent, CalendarDay, TimeBlock, Action, WorkSession, Observation, Belief, Fact, Artifact, Experience, Outcome, Meeting, Interaction, Measurement

### POLICY Domain (8 types)
Guardrail, Principle, Process, System, Proposal, Alert, Insight, Pattern

### AUDIT Domain (6 types)
AuditLogEntry, EvidenceLogEntry, EventLogEntry, GraphOp, ActionOp, ActionResult

---

## 7. RELATIONSHIP TYPES (56 total)

**By Layer:**
- Foundation: HOLDS_VALUE, FOLLOWS_PRINCIPLE, RESPECTS_GUARDRAIL, DEFINED_BY, PARTNERS_WITH, PROTECTED_BY, RELATES_TO, OPERATES_IN, SUPPORTED_BY, ENFORCED_BY, ALIGNS_WITH
- Vision: ENABLES, HAS_GOAL, HAS_METRIC, PURSUES_GOAL, HOLDS_EQUITY, ROLE_IN
- Strategy: SERVES, EXECUTES, HAS_KEY_RESULT, MEASURES, CONTAINS, ADVANCES, HAS_MILESTONE, HAS_RISK
- Tactics: HAS_PROJECT, DELIVERS, HAS_PROCESS, HAS_TASK, GENERATES_TASK, USES, DEPENDS_ON, SUPPORTS
- Execution: SCHEDULED_AS, LOGGED_AS, AUTHORIZES, BASED_ON
- Track: TRACKS, OBSERVES, INFORMS
- Mind: APPLIED_TO, TRIGGERS, VALIDATES, DEMONSTRATES, SHAPED_BY, EXHIBITS, AGGREGATES, INFORMS_IDENTITY
- Policy: DERIVED_FROM, CHANGES, GOVERNS

---

## 8. QUESTIONS FOR GPT-5.2-Pro

1. **Domain-First vs Layer-First:** Should we replace layer filters with domain filters, or keep both?

2. **Loop Phase Integration:** How should loop phases appear in World Model view? As a filter? As a visual grouping?

3. **Integrity Badges:** Should violations show as badges on nodes, or as a separate filter panel?

4. **Filter Persistence:** Should filter state persist across sessions? Per-view or global?

5. **Mobile/Touch:** How to handle 60+ filter elements on smaller screens?

6. **Performance:** With domain filtering, should we pre-compute domain membership or compute on-the-fly?

7. **Journey Mode:** Should journey path-finding respect domain boundaries or cross them?

8. **Default State:** What should be visible by default? All domains? Only active items?

---

*Generated by ODEI Symbiosis — Claude (AI Principal) + Anton (Human Physical Layer)*
