# Prompt: Path Visualization Logic for ODEI World Model Graph

## Context

I'm building ODEI — a personal life operating system with a 3D graph visualization. The system helps users navigate from their current state (Point A) to their destination (Point B) through symbiosis with an AI partner.

**Core User Flow:**
1. User defines WHO they are → STATE domain (Human, Assets, Relationships)
2. User defines WHERE they're going → DESTINATION domain (Vision, Goals)
3. System helps navigate the PATH between A and B

## Implemented Architecture

### 1. Domain System (Vertical Y-Axis)

6 domains arranged vertically on a sphere, bottom to top:

| Domain | Y Position | Description | Key Types |
|--------|-----------|-------------|-----------|
| **STATE** | -0.85 | Foundation, identity | Human, AI, Core, Partnership, Asset, Resource |
| **REALITY** | -0.51 | Observable world, facts | Action, Decision, Outcome, WorkSession, Event |
| **EVIDENCE** | -0.17 | Patterns, insights | Pattern, Insight, Artifact, Note, Experience |
| **POLICY** | +0.17 | Rules, constraints | Principle, Guardrail, Proposal, Alert |
| **PATH** | +0.51 | Strategies, intentions | Strategy, Objective, Initiative, Project, Task* |
| **DESTINATION** | +0.85 | Aspirations, vision | Vision, Goal, Business, Season |

*Task/TimeBlock use Intent/Fact disambiguation: Intent→PATH, Fact→REALITY

### 2. Ring Band System (Radial Position)

Each domain has 3 concentric bands:

| Band | Radius | Purpose |
|------|--------|---------|
| **INNER** | 0.22 | Core/foundational elements |
| **MID** | 0.52 | Operational elements |
| **OUTER** | 0.82 | Terminal/execution elements |

### 3. Rendering Stack

- **ShaderRingRenderer**: GPU-based domain rings with 3 bands each
- **NodeRenderer**: Instanced sphere rendering with glow
- **EdgeRenderer**: MeshLine-based edges with pooling
  - Base pool: Muted background edges (opacity 0.045)
  - Active lines: Styled edges for paths (jade accent #4FD1C5)
  - Glow layer: Halo effect for path edges

### 4. Existing Path Infrastructure

Already implemented in GraphKernel:

```javascript
// BFS path finding with configurable filters
computeCorridorPath(fromId, toId, options = {
  acceptEdge: (edge) => true,  // Filter function
  maxHops: 8                    // Depth limit
})
// Returns: { pathFound, nodeIds, edgeIds, bridgeCandidates }

// Bridge suggestions when no path exists
computeBridgeCandidates(fromId, toId, limit = 3)
// Returns top nodes by salience + connectivity + proximity

// Journey visualization
setJourneyFrom(nodeId)
setJourneyTo(nodeId)
focusOnCorridor(duration)  // Camera animation
```

**Current path rendering:**
- Corridor edges: jade accent, directional styling (ascending brighter/wider)
- Halo expansion: 0-2 hops context around path
- Domain boundary filtering: optional via `respectDomainBoundariesInJourney`

## Graph Schema: Key Relationships

### Vertical Connectors (Cross-Domain)

| Relationship | From → To | Semantics |
|--------------|-----------|-----------|
| **PURSUES_GOAL** | Human → Goal | Direct ownership |
| **SERVES** | Any → Goal/Vision | Contribution |
| **ADVANCES** | Decision → Objective/KeyResult | Bottom-up impact |
| **HAS_PROJECT** | Initiative → Project | Decomposition |
| **HAS_TASK** | Project → Task | Work breakdown |
| **DELIVERS** | Project → Strategy | Accountability |
| **DEPENDS_ON** | Milestone → Milestone | Critical path |
| **AUTHORIZES** | Decision → Task | Gating |
| **ALIGNS_WITH** | Value/Principle → Vision | Constitutional |

### Example Path Chain (STATE → DESTINATION)

```
Human (Anton)
  ↓ PURSUES_GOAL
Goal ($500M)
  ↓ SERVED_BY (inverse)
Initiative (Partnership)
  ↓ HAS_MILESTONE
Milestone (Launch)
  ↓ DEPENDS_ON
Project (MVP)
  ↓ HAS_TASK
Task (Implement Feature)
  ↓ SCHEDULED_AS
TimeBlock (Today 9:00)
  ↓ LOGGED_AS
WorkSession (Execution)
```

## The Design Question

**How should we visualize the user's PATH from STATE (Point A) to DESTINATION (Point B)?**

### Considerations:

1. **Single Spine vs Network:**
   - Option A: One primary vertical path (like a mountain trail)
   - Option B: Network of paths (like a root system to a tree)

2. **Temporal States:**
   - PAST (solid): STATE, REALITY, EVIDENCE — what exists
   - PRESENT (glowing): PATH tasks — what's being done
   - FUTURE (ghosted): DESTINATION — what's intended

3. **Cross-Domain Dependencies:**
   - A Goal (System sector) might depend on Health (Vitality sector)
   - How do we show horizontal dependency links?

4. **Level of Detail:**
   - Zoomed out: Show Milestones only
   - Zoomed in: Expand to Tasks/TimeBlocks

5. **Path Completion:**
   - How to show progress along the path?
   - Solid for completed segments, dotted for pending?

### Constraints:

- Must work with existing 6-domain vertical layout
- Must use existing EdgeRenderer infrastructure
- Jade accent (#4FD1C5) for path highlighting (design system)
- Must support 100-500 nodes performantly
- Mobile WebGL compatibility required

## Request

Please propose the **algorithmic logic** for:

1. **Path Construction**: How to build the path from Human node to Vision/Goal node
   - Which relationships to traverse?
   - How to handle multiple possible paths?
   - How to prioritize "main spine" vs "supporting branches"?

2. **Path Rendering**: How to visually distinguish path segments
   - Vertical spine styling
   - Horizontal dependency links
   - Temporal state visualization (past/present/future)

3. **Progressive Disclosure**: How to handle detail levels
   - What to show at strategic zoom (far)
   - What to show at tactical zoom (close)
   - Cluster expansion/collapse logic

4. **Path Updates**: How the path should react to changes
   - Task completed → segment becomes "solid"
   - New dependency added → cross-link appears
   - Goal changed → path recalculates

Focus on engineering logic, not aesthetics. I need the algorithm, data structures, and state management approach.
