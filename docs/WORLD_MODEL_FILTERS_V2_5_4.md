# World Model Filters v2.5.4

## Overview
This document describes the domain-first, loop-phase-first filtering system for Atlas (World Model) aligned with ODEI v2.5.4. Filters act as **lenses**: they affect visibility/dimming/budgets only and **do not** recompute layout or move nodes.

## Filter Hierarchy
Primary -> Advanced:
1) **Domains** (STATE, DESTINATION, PATH, REALITY, POLICY, AUDIT)
2) **Loop Phases** (OBSERVE, DECIDE, ACT, VERIFY, EVOLVE)
3) **Status Groups** (ACTIVE, BLOCKED, TERMINAL)
4) **Integrity** (guard violations + severity)
5) **Advanced** (Layers -> Types -> Relationships -> ROI -> Date -> Time Window -> Complexity -> Show edges -> Journey options)

## Defaults (v2.5.4)
- **Domains ON:** DESTINATION, PATH, STATE, REALITY
- **Domains OFF:** POLICY, AUDIT
- **Loop Phases:** all ON
- **Status ON:** ACTIVE + BLOCKED; **OFF:** TERMINAL
- **Integrity:** `Only show issues` OFF; all severities enabled when Integrity filter is ON
- **Journey:** `Respect domain boundaries` OFF

### Audit Mode Behavior
- Entering **Verify** mode auto-enables the **AUDIT** domain if it was off.
- Leaving Verify restores AUDIT to OFF if it was auto-enabled and the user did not override it.

## Domain Mapping
Nodes are assigned a domain during normalization:
- If a valid `domain` property exists on the node, it is used.
- Otherwise, map from `typeNormalized` using canonical TYPE -> DOMAIN mapping.
- Ambiguous types (e.g., Guardrail / Principle / Process / System / Policy) are disambiguated using layer when available.
- If mapping fails, default to **PATH** (log a warning in dev).

### Canonical TYPE -> DOMAIN (selected)
- **STATE:** Human, Core, AI, Identity, Capability, Resource, Constraint, HealthSnapshot, Position, Context, Metric, Signal, Value, Principle, Guardrail, Asset, Policy, Partnership, Relationship
- **DESTINATION:** Vision, Business, Goal, Season, Value, NorthStar, KeyResult, Area
- **PATH:** Strategy, Objective, Initiative, Project, Path, Task, Milestone, Decision, Risk, TimeBlockIntent, System, Process, Program, Roadmap, Opportunity
- **REALITY:** Person, Organization, CalendarEvent, CalendarDay, TimeBlock, Action, WorkSession, Observation, Belief, Fact, Artifact, Experience, Outcome, Meeting, Interaction, Measurement
- **POLICY:** Guardrail, Principle, Process, System, Proposal, Alert, Insight, Pattern
- **AUDIT:** AuditLogEntry, EvidenceLogEntry, EventLogEntry, GraphOp, ActionOp, ActionResult

## Loop Phase Inference
If `loopPhase` is missing, infer by type:
- **OBSERVE:** Observation, Metric, Signal, Measurement, Meeting, Interaction, EvidenceLogEntry, EventLogEntry
- **DECIDE:** Decision, Proposal, Objective, KeyResult, Risk, Strategy
- **ACT:** Task, Action, WorkSession, TimeBlockIntent, TimeBlock, CalendarEvent, Outcome
- **VERIFY:** Measurement, Observation, EvidenceLogEntry, EventLogEntry
- **EVOLVE:** Insight, Pattern, Lesson, Note, Document, Evidence

If uncertain, `loopPhase` remains undefined and **does not exclude** nodes by default.

## Integrity (Guard Queries)
Integrity guard violations are computed per node and surfaced in the **Integrity** filter:
- **A** ORPHAN_TASK (HIGH)
- **B** INTENT_MISSING_TASK (HIGH)
- **C** PROPOSAL_INVALID_OPS (MEDIUM)
- **D** ACTION_NO_COMPLETES (HIGH)
- **E** WORKSESSION_MISSING_BOUNDS (HIGH)
- **F** ENTITY_MISSING_ID (CRITICAL)
- **G** ENTITY_TYPE_MISMATCH (HIGH)
- **H** DECISION_NO_OBSERVATION_BASIS (HIGH)
- **I** PROPOSAL_NO_SOURCE_LINEAGE (MEDIUM)
- **J** GRAPHOP_UNAUDITED (CRITICAL)

Renderer behavior:
- **CRITICAL/HIGH** show subtle badges on visible nodes (no flashing).
- **MEDIUM/LOW** are surfaced in tooltips and counts.

If backend guard results exist, prefer them; otherwise compute best-effort client-side from available properties/relationships.

## Renderer Visibility Pipeline (v2.5.4)
1) Domain filter
2) Loop phase filter
3) Status group filter
4) Layer filter (Advanced)
5) Type filter (Advanced)
6) Relationship filter
7) Value filters (ROI / search / date / time window)
8) Integrity filter (if `integrityOnly`)
9) Forced nodes (selected/pinned/focusPath) always included
10) Journey corridor expansion (journey mode only)
11) Budget fill by salience
12) Edge/Label final policies

## Persistence
Filter state persists **per view, per workspace, per user**:
- Key: `odei.worldModel.filters.{workspaceId}.{userId}.v2_5_4`
- Migration: legacy layer/type state retained; domains/loop/status default to v2.5.4.

## Search Fields
Search matches combined text across:
- title / summary / description
- type / layer / domain / loopPhase

## Performance Notes
- Domain/loop/status/layer membership is precomputed in `normalizeGraph()`.
- Bitmask membership enables fast visibility checks.
- Filter recompute happens only on filter/data change, not per frame.
- Visibility only: no layout recompute or position teleporting.
