# World Model Visualization System

> Deterministic 6-domain graph visualization for ODEI World Model
> Implemented: January 2026

## Overview

The World Model Visualization system renders graph nodes across 6 semantic domains arranged vertically on a sphere, with nodes positioned deterministically using FNV-1a hashing for stable angular placement and type-based radial band assignment.

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    MonumentalGraph (index.js)                    │
│                         ┌─────────────┐                          │
│                         │  Renderer   │                          │
│                         └──────┬──────┘                          │
│                                │                                 │
│         ┌──────────────────────┼──────────────────────┐          │
│         ▼                      ▼                      ▼          │
│  ┌─────────────┐      ┌───────────────┐      ┌─────────────┐    │
│  │ NodeRenderer│      │ShaderRingRend.│      │ EdgeRenderer│    │
│  └─────────────┘      └───────┬───────┘      └─────────────┘    │
│                               │                                  │
│                               ▼                                  │
│                    ┌─────────────────────┐                       │
│                    │  armillaryLayout.js │                       │
│                    └──────────┬──────────┘                       │
│                               │                                  │
│              ┌────────────────┼────────────────┐                 │
│              ▼                ▼                ▼                 │
│    ┌─────────────────┐ ┌─────────────┐ ┌────────────────┐       │
│    │domainResolver.js│ │ringResolver │ │WorldModelTuning│       │
│    └─────────────────┘ └─────────────┘ └────────────────┘       │
└─────────────────────────────────────────────────────────────────┘
```

## Domain Classification

### 6 Domains (Bottom to Top by Y)

| Domain | Y Position | Description | Example Types |
|--------|-----------|-------------|---------------|
| **STATE** | -0.75 | Foundation, identity | Human, AI, Core, Partnership |
| **REALITY** | -0.45 | Observable world, facts | Action, Decision, Outcome, Event |
| **EVIDENCE** | -0.15 | Patterns, insights | Pattern, Insight, Artifact, Note |
| **POLICY** | +0.15 | Rules, constraints | Principle, Guardrail, Proposal |
| **PATH** | +0.45 | Strategies, intentions | Strategy, Objective, Project, Task* |
| **DESTINATION** | +0.75 | Aspirations, vision | Vision, Goal, Business, Season |

*Task/TimeBlock/TimeBlockIntent use Intent/Fact disambiguation

### Classification Algorithm

```javascript
resolvePrimaryDomain(node):
  1. DESTINATION if type in {Vision, Business, Goal, Season, Value}
  2. POLICY if type in {Principle, Guardrail, Proposal, Alert, Policy}
  3. EVIDENCE if type in {Insight, Pattern, Note, Source, Artifact, ...}
  4. PATH if type in {Strategy, Objective, KeyResult, Initiative, ...}
  5. For Task/TimeBlock/TimeBlockIntent:
     - PATH if has Intent label OR non-terminal status
     - REALITY if has Fact label OR terminal status
  6. REALITY if type in {Action, WorkSession, Decision, Outcome, ...}
  7. STATE if type in {Human, Core, Context, AI, Mind, ...}
  Default: STATE
```

### Intent/Fact Disambiguation

Types that can exist in multiple domains based on lifecycle state:
- `Task`, `TimeBlock`, `TimeBlockIntent`

**Rules:**
- **Intent label** OR **non-terminal status** → PATH domain (active intention)
- **Fact label** OR **terminal status** → REALITY domain (completed)

Terminal statuses: `completed`, `cancelled`, `canceled`, `archived`, `done`, `failed`

## Ring Band System

### 3 Concentric Bands per Domain

Each domain disk has 3 radial bands for hierarchical positioning:

| Band | Radius Fraction | Opacity | Purpose |
|------|-----------------|---------|---------|
| **INNER** | 0.22 | 0.12 | Core/foundational elements |
| **MID** | 0.52 | 0.08 | Operational/tactical elements |
| **OUTER** | 0.82 | 0.05 | Terminal/execution elements |

### Band Assignment by Domain

**DESTINATION:**
- INNER: Vision
- MID: Business, Goal, Value
- OUTER: Season

**PATH:**
- INNER: Strategy, Objective, KeyResult, Initiative, Milestone, Risk
- MID: Area, Project, System, Process
- OUTER: Task*, TimeBlock*, TimeBlockIntent*

**POLICY:**
- INNER: Principle, Guardrail
- MID: Policy
- OUTER: Proposal, Alert

**EVIDENCE:**
- INNER: Pattern, Insight, Belief
- MID: Artifact, Experience, Evidence
- OUTER: Note, Source, AuditLogEntry, GraphOp, ...

**REALITY:**
- INNER: Decision, Outcome, Fact
- MID: Action, WorkSession, Execution, Event
- OUTER: Observation, CalendarEvent, CalendarDay, Task*, ...

**STATE:**
- INNER: Human, Core, Context, AI, Mind, Partnership, Foundation
- MID: Asset, Resource, Relationship
- OUTER: Metric, Signal, Person, Organization

## Stable Angular Positioning

### FNV-1a Hash Algorithm

Nodes are positioned deterministically using FNV-1a hash of their ID:

```javascript
function stableAngle(nodeId):
  hash = FNV-1a(nodeId)  // 32-bit unsigned
  angle = (hash / 0xFFFFFFFF) * 2π
  return angle  // radians, 0 to 2π
```

**Properties:**
- Same ID always produces same angle
- Well-distributed across angular range
- No random component - purely deterministic

## ShaderRingRenderer

GPU-based ring rendering using fragment shaders:

### Features
- Mathematically perfect circles (no vertex artifacts)
- 3 concentric bands per domain with configurable opacity
- Sector support for district visualization
- Hover/selection highlighting with jade accent
- Anti-aliased edges using `fwidth()` and `smoothstep()`

### Shader Uniforms
```glsl
uniform float uRadius;        // Domain ring radius
uniform vec3 uBandRadii;      // [INNER, MID, OUTER] as fractions
uniform vec3 uBandOpacities;  // Per-band opacity
uniform vec4 uSectors[12];    // Sector angle ranges
uniform int uSectorCount;     // Active sector count
uniform int uHoveredSector;   // -1 = none
```

### Safety Measures
- `precision highp float` for mobile compatibility
- Epsilon guard on `atan()` to prevent undefined behavior at origin
- Early discard for transparent fragments

## File Structure

```
src/
├── modules/worldmodel/visualization/
│   ├── domainResolver.js     # Node → Domain classification
│   └── ringResolver.js       # Node → Ring band assignment
│
└── Renderer/MonumentalGraph/
    ├── index.js              # Main orchestrator
    └── worldmodel/
        ├── config/
        │   └── WorldModelTuning.js  # Layout parameters
        ├── layout/
        │   └── armillaryLayout.js   # Position computation
        └── render/
            └── ShaderRingRenderer.js # GPU ring rendering
```

## Type Coverage

All 56 node types from SCHEMA.md are classified:

| Domain | Count | Types |
|--------|-------|-------|
| DESTINATION | 5 | vision, business, goal, season, value |
| POLICY | 5 | principle, guardrail, proposal, alert, policy |
| EVIDENCE | 16 | insight, pattern, note, source, artifact, experience, evidence, belief, audit_log_entry, graph_op, action_op, evidence_log_entry + variants |
| PATH | 11 | strategy, objective, key_result, initiative, milestone, risk, area, project, system, process + variants |
| INTENT_FACT | 5 | task, time_block, time_block_intent + variants |
| REALITY | 13 | action, work_session, decision, observation, calendar_event, calendar_day, outcome, execution, event, fact + variants |
| STATE | 14 | human, core, context, asset, resource, metric, signal, person, organization, relationship, ai, mind, partnership, foundation |

## API

### domainResolver.js

```javascript
// Classify node into domain
resolvePrimaryDomain(node) → 'state'|'reality'|'evidence'|'policy'|'path'|'destination'

// Get Y coordinate for domain
getDomainY(domain, graphRadius) → number

// Get classification explanation
getDomainReason(node) → string

// Check if type is unknown
isUnclassified(node) → boolean

// Batch classify nodes
classifyNodes(nodes) → { classified: Map, unclassified: [], stats: {} }
```

### ringResolver.js

```javascript
// Assign node to radial band
resolveRingBand(node, domain) → 'INNER'|'MID'|'OUTER'

// Get radius fraction for band
getBandRadius(band) → 0.22|0.52|0.82

// Deterministic angle from ID
stableAngle(nodeId) → number (0 to 2π)

// Get all band radii
getAllBandRadii() → [{band, radius}, ...]
```

### ShaderRingRenderer

```javascript
// Update rings from layout
update(layout, tuning)

// Set global opacity multiplier
setOpacity(opacity)

// Set individual band opacities
setBandOpacities(inner, mid, outer)

// Toggle domain labels
setLabelsVisible(visible)

// Set hover state
setHoveredSector(domain, sectorIndex)

// Animation tick
tick(deltaTime)

// Cleanup
dispose()
```

## Configuration

### WorldModelTuning.js

Key parameters:
```javascript
{
  domainBands: {
    state:       { y: -0.75, thickness: 0.12 },  // Bottom: Foundation
    reality:     { y: -0.45, thickness: 0.18 },  // Observable world
    evidence:    { y: -0.15, thickness: 0.14 },  // Patterns, insights
    policy:      { y:  0.15, thickness: 0.08 },  // Rules, guardrails
    path:        { y:  0.45, thickness: 0.22 },  // Strategies, routes
    destination: { y:  0.75, thickness: 0.05 },  // Top: Vision beacon
  },
  radius: { base: 360, min: 360, max: 1000 },
  jitter: { y: 0.06, radial: 0.014, theta: 0.015 },
}
```

## Journey System

### Overview

Journey = **Spine** + **Branches** + **Halo**

- **Spine**: Main actionable corridor from STATE to DESTINATION
- **Branches**: Supporting connections (dependencies, policy, evidence, risk)
- **Halo**: 0-2 hop context around spine

### Why Not Shortest Path?

BFS shortest path produces useless corridors like `Human → PURSUES_GOAL → Goal`.
We need **K-candidates with scoring** to find the "best actionable path".

### Path Scoring Algorithm

```javascript
score = -Σ edgeCost - 0.35 * hops
      + 6.0 * activeIntentNodes      // Reward active work
      + 4.0 * hasPATHDomain          // Require strategic nodes
      + 2.0 * hasExecutionNearNow    // Reward current activity
      - 3.0 * blockedSegments        // Penalize blockers
      - 50  * noPathDomain (if dest=DESTINATION)  // Heavy penalty for trivial paths
```

### Edge Classifications

**Corridor Edges (Spine traversal):**
- Ownership: PURSUES_GOAL, HAS_GOAL, HAS_VISION
- Contribution: SERVES, ADVANCES, DELIVERS
- Decomposition: HAS_PROJECT, HAS_TASK, HAS_MILESTONE
- Gating: DEPENDS_ON, AUTHORIZES

**Branch Edges:**
- Dependency: DEPENDS_ON, SUPPORTS, CONFLICTS_WITH
- Policy: RESPECTS_GUARDRAIL, ENFORCED_BY, ALIGNS_WITH
- Evidence: BASED_ON, INFORMS, VALIDATES

### Progress States

| State | Description | Visual |
|-------|-------------|--------|
| COMPLETED | Fact label, terminal status | Solid |
| ACTIVE | IN_PROGRESS, today's TimeBlock | Glowing |
| PENDING | READY, BACKLOG, Intent | Semi-transparent |
| BLOCKED | Has blocking dependency | Dimmed + indicator |

### LOD (Level of Detail)

| LOD | Distance | Visible Types |
|-----|----------|---------------|
| STRATEGIC | >400 | Vision, Goal, Strategy, Milestone, Principle |
| TACTICAL | 150-400 | + Task, Project, Risk, Alert |
| EXECUTION | <150 | + TimeBlock, WorkSession, Observation |

### API

```javascript
import { JourneyStore, computeJourney, PROGRESS_STATE, LOD } from './journey';

// Create store
const journeyStore = new JourneyStore(kernel);

// Set journey
journeyStore.setJourney(humanNodeId, goalNodeId);

// Get state
const { spine, branches, halo, completion, nextActionNode } = journeyStore.getState();

// Check membership
journeyStore.isSpineNode(nodeId);
journeyStore.isSpineEdge(edgeId);
journeyStore.getBranchType(nodeId); // 'dependency'|'policy'|'evidence'|'risk'|null

// LOD control
journeyStore.setLODFromDistance(cameraDistance);
journeyStore.isVisibleAtLOD(nodeId);
```

### MonumentalGraph Integration

The JourneyStore is integrated into MonumentalGraph for seamless journey visualization:

```javascript
// JourneyStore is created automatically in MonumentalGraph constructor
const graph = new MonumentalGraph(container, { layoutMode: 'armillary' });

// Set journey endpoints
graph.setJourneyFrom(humanNodeId);
graph.setJourneyTo(goalNodeId);

// Access journey state
const state = graph.getJourneyState();
const spineNodes = graph.getSpineNodeIds();
const branchType = graph.getBranchType(nodeId); // 'dependency'|'policy'|'evidence'|'risk'|null
const progress = graph.getNodeProgress(nodeId); // 'COMPLETED'|'ACTIVE'|'PENDING'|'BLOCKED'
const completion = graph.getJourneyCompletion(); // 0-1
const nextAction = graph.getNextActionNode();

// Subscribe to journey changes
const unsubscribe = graph.onJourneyStateChange((state) => {
  console.log('Journey updated:', state.completion);
});

// LOD updates automatically based on camera distance
// Or manually get current LOD
const lod = graph.getCurrentLOD(); // 'STRATEGIC'|'TACTICAL'|'EXECUTION'
```

### EdgeRenderer Journey Styles

| Style | Use Case | Visual |
|-------|----------|--------|
| `path` | Spine edges | Jade accent, glow, progress-based width |
| `branch` | Branch edges (dep/policy/evidence/risk) | Colored by type, thinner, no glow |
| `halo` | Context edges | Muted silver, base pool |
| `muted` | Background edges | Very subtle silver |

### Progress-Based Edge Styling

| Progress | Edge Color | Width | Glow |
|----------|------------|-------|------|
| COMPLETED | Bright jade `#6feee5` | +10% | +30% |
| ACTIVE | Standard jade `#4fd1c5` | Normal | Normal |
| PENDING | Dimmer jade `#3db8ad` | -15% | -40% |
| BLOCKED | Gray `#9ca3af` | -30% | -60% |

### Branch Type Colors

| Branch Type | Color | Description |
|-------------|-------|-------------|
| dependency | Cool gray `#94a3b8` | DEPENDS_ON, SUPPORTS edges |
| policy | Warm muted `#b8a394` | RESPECTS_GUARDRAIL, GOVERNS edges |
| evidence | Green-gray `#a3b894` | BASED_ON, INFORMS edges |
| risk | Red-gray `#b89494` | HAS_RISK, MITIGATED_BY edges |

## Implementation Status

| Feature | Status | Notes |
|---------|--------|-------|
| 6-Domain Layout | ✅ Complete | STATE→DESTINATION vertical arrangement |
| 3-Band Ring System | ✅ Complete | INNER/MID/OUTER radial positioning |
| ShaderRingRenderer | ✅ Complete | GPU-based, mobile compatible |
| JourneyPlanner | ✅ Complete | K-shortest paths with scoring |
| JourneyStore | ✅ Complete | State management, LOD, hysteresis |
| Progress States | ✅ Complete | COMPLETED/ACTIVE/PENDING/BLOCKED |
| Branch Visualization | ✅ Complete | dependency/policy/evidence/risk |
| LOD System | ✅ Complete | STRATEGIC/TACTICAL/EXECUTION |
| EdgePolicy Integration | ✅ Complete | Branch/halo styles |
| EdgeRenderer Styles | ✅ Complete | Progress-based coloring |

## Future Extensions

1. **Sector Visualization** - District-based angular grouping (infrastructure ready)
2. **Journey Animation** - Smooth spine reveal animation (infrastructure ready)
3. **Cluster Aggregation** - Virtual aggregate nodes at LOD_STRATEGIC
4. **Path Alternatives UI** - Switch between alternative spines (data ready)

---

*Last updated: January 2026*
*Implementation: Claude (AI Principal, ODEI Symbiosis)*
