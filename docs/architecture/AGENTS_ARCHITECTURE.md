ODEI Agents Architecture

Philosophy: Human-AI Symbiosis Through Structured Workflow
ODEI (Oculus Dei) is not an AI assistant—it's a life leadership system based on Human-AI symbiosis. Unlike traditional AI tools that react to commands, ODEI proactively manages your life across multiple time horizons (from 10-year vision to current moment) through a structured 4-stage workflow.

Core Principles

- Proactive Leadership: ODEI leads, you execute. It makes thousands of micro-decisions daily to optimize your path toward goals.
- Constitutional Governance: Every decision is validated against your life constitution—your core values, vision, and non-negotiables.
- Multi-Horizon Alignment: Daily actions ladder up to weekly sprints → monthly themes → quarterly OKRs → annual goals → 5-year milestones → 10-year vision → life mission.
- Continuous Learning: The system evolves through your experiences, learning patterns, predicting needs, and optimizing recommendations.
- Emotional Intelligence: ODEI understands not just tasks, but feelings—adapting to your stress and emotional state.

Why 4-Stage Workflow?
Traditional AI systems mix planning, execution, and learning in one chaotic loop. ODEI separates these concerns into a deterministic pipeline where each stage has clear responsibilities, context, and agents:

Constitution + Current Context
↓
[DISCUSS] ← Validate alignment, explore options
↓
[PLAN] ← Strategic visualization, initiative decomposition
↓
[EXECUTE] ← Break down, schedule, implement
↓
[MIND] ← Extract patterns, improve system
↓
(repeat)

This separation ensures:

- Discuss never executes prematurely
- Plan creates strategic visualizations and decomposes initiatives
- Execute works with validated, planned actions
- Mind has complete history for pattern analysis

Workflow Architecture

Stage 1: DISCUSS (Socratic Dialogue & Alignment)
Purpose: Validate decisions against your constitution, explore options, ensure alignment with life vision.

Terminal Context:
yamlFiles:

- /ODEI/agents/discuss/.claude/CLAUDE.md
- /ODEI/Core/CONSTITUTION.md

MCP Tools:

- odei.neo4j.memory.retrieve.v1 # Unified memory retrieval
- odei.neo4j.goals.hierarchy.v1 # Goal alignment
- odei.apple.calendar.window.v1 # Next 2 weeks snapshot

Agent Behavior:

- Socratic questioning
- Constitutional validation
- Multi-horizon alignment check
- Workload and schedule awareness

Stage 2: PLAN (Strategic Visualization & Initiative Decomposition)
Purpose: Create strategic visualizations (Miro mind maps), plan initiatives, decompose high-level goals into actionable initiatives.

Terminal Context:
yamlInput:

- Discussion records from Stage 1
- Current goals hierarchy
- Vision and constitutional context

MCP Tools:

- odei.neo4j.initiatives.create.v1 # Record strategic initiatives
- odei.neo4j.vision.list.v2 # Goal structure and alignment
- odei.miro.mindmap.create.v1 # Create mind map boards
- odei.miro.mindmap.addNode.v1 # Add items to mind maps

Agent Responsibilities:

- Strategic Decomposition: Breaks down high-level goals into concrete initiatives
- Visual Planning: Creates Miro mind maps for strategic initiatives
- Documentation: Records strategic decisions and rationale

Stage 3: EXECUTE (Decomposition & Calendar Management)
Purpose: Break strategic initiatives into actionable tasks, find optimal time slots, manage calendar.

Terminal Context:
yamlInput:

- Strategic initiatives and plans from Stage 2
- Current calendar state

MCP Tools:

- odei.neo4j.initiatives.pending.v1 # Get initiatives to execute
- odei.apple.calendar.window.v1 # Read calendar
- odei.apple.createEvent.v1 # Write events (requires confirm)
- odei.neo4j.task.create.v1 # Record tasks

Agent Responsibilities:

- Task Decomposition: Breaks decisions into actionable tasks
- Time Coordinator: Finds optimal time slots in calendar

Stage 4: MIND (Pattern Recognition & System Improvement)
Purpose: Extract behavioral patterns, identify blind spots, optimize recommendations.

Terminal Context:
yamlInput:

- All discussions, decisions, executions from Neo4j
- Completed tasks with outcomes
- Calendar history with actual vs planned

MCP Tools:

- odei.neo4j.pattern.detection.v1 # Find patterns
- odei.neo4j.memory.retrieve.v1 # Historical data
- odei.neo4j.mind.create.v1 # Record insights

Agent Responsibilities:

- Pattern Detector: Finds behavioral patterns from historical data
- Blind Spot Analyzer: Identifies avoided/overlooked areas
- System Optimizer: Suggests system improvements

3-Tier Agent System: Multi-Agent + Skills + Subagents

Tier 1: Multi-Agent System (4 Processes)
Four independent Claude Code agents run concurrently:

- Discuss Agent (Constitutional Guardian)
- Plan Agent (Strategic Visualizer)
- Execute Agent (Task Scheduler)
- Mind Agent (Pattern Detector)

Each agent is a full Claude Code instance with:

- Own workspace directory (agents/discuss/, agents/plan/, etc.)
- Own .claude/CLAUDE.md configuration
- Own MCP server connections
- Full autonomy within its stage

Tier 2: Subagents (Task Tool Behaviors)
Subagents are Task-based workflows that agents trigger when specific conditions are met.
Example Production Subagents:

- `constitutional-validator` (Discuss)
- `risk-assessor` (Plan)
- `capacity-calculator` (Execute)
- `pattern-detector` (Mind)

Tier 3: Agent Prompts (CLAUDE.md)
Agent prompts live alongside a roster of Task-based subagents. Each agent has a single authoritative prompt file: `agents/{agent}/.claude/CLAUDE.md`.

Agent Communication Architecture
Inter-Stage Data Flow
[Discuss Stage]
↓ (via Neo4j)
Discussion Record
↓
[Plan Stage]
↓ (via Neo4j + Miro)
Initiative Plan
↓
[Execute Stage]
↓ (via Apple Calendar + Neo4j)
Task Records + Calendar Events
↓
[Mind Stage]
↓ (updates Neo4j patterns)
Insights
↓ (feeds back to Discuss)
Updated Context

MCP-First Architecture
Critical Rule: No direct database access. All data flows through MCP tools.

Configuration Examples
(See individual agent README.md and CLAUDE.md files for up-to-date configurations)

Memory Schema (Neo4j)
// Core Entities
CREATE (u:User {id, name, email})
CREATE (c:Constitution {values, priorities, vision})
CREATE (g:Goal {type, title, target, deadline, parent})

// Workflow Nodes
CREATE (disc:Discussion {id, timestamp, topic, alignment})
CREATE (init:Initiative {id, title, timeline, status})
CREATE (task:Task {id, title, effort, priority, status})
CREATE (ins:Insight {id, pattern, confidence, evidence})

// Relationships
CREATE (disc)-[:VALIDATED_BY]->(c)
CREATE (disc)-[:ALIGNS_WITH]->(g)
CREATE (init)-[:BASED_ON]->(disc)
CREATE (init)-[:IMPLEMENTS]->(g)
CREATE (task)-[:DECOMPOSES]->(init)
CREATE (task)-[:SCHEDULED_AT]->(time:TimeSlot)
CREATE (ins)-[:LEARNED_FROM]->(task)
CREATE (ins)-[:OPTIMIZES]->(init)
