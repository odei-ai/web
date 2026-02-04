# ODEI Agent Handoff Protocol

## Overview

The ODEI system uses a structured 4-stage workflow where agents hand off work to each other through **control frames** and **Neo4j data structures**. This ensures deterministic, traceable flow and prevents chaos.

## Control Frame Format

Control frames are special markers in terminal output that signal system events.

### Format

```
<<<ODEI:[TYPE]:[PAYLOAD]>>>
```

### Types

#### 1. READY Frame

Emitted when an agent initializes successfully.

```
<<<ODEI:READY:{"stage":"discuss","ts":"2025-10-09T18:30:00Z"}>>>
```

**Fields**:

- `stage`: Agent name (discuss | decisions | execute | mind)
- `ts`: ISO8601 timestamp of initialization

**Purpose**: Signals to Electron app that agent is ready to receive input.

#### 2. HANDOFF Frame

Emitted when an agent completes its work and hands off to next stage.

```
<<<ODEI:HANDOFF:decisions>>>
```

**Payload**: Target stage name (decisions | execute | mind)

**Purpose**: Triggers the Electron app to switch focus to the next agent terminal.

#### 3. ERROR Frame

Emitted when an agent encounters a fatal error.

```
<<<ODEI:ERROR:{"code":"MCP_CONNECTION_FAILED","message":"Cannot connect to odei-neo4j"}>>>
```

**Fields**:

- `code`: Error code (uppercase snake_case)
- `message`: Human-readable error description

**Purpose**: Alerts user and Electron app to failures.

## Workflow Stages

### Stage 1: Discuss → Decisions

**Discuss Agent**:

1. Validates proposal against constitution
2. Performs Socratic exploration
3. Checks goal alignment
4. Records discussion in `odei.history.threads`
5. Emits: `<<<ODEI:HANDOFF:decisions>>>`

**Data Passed**:

- Discussion thread ID (in odei-history SQLite)
- Constitutional check results
- Goal alignment score
- Recommendation (APPROVE | REJECT | REVISE)

**Decisions Agent**:

1. Receives handoff signal
2. Reads discussion thread from odei-history
3. Begins ROI calculation

---

### Stage 2: Decisions → Execute

**Decisions Agent**:

1. Calculates ROI for all options
2. Ranks by total ROI
3. Presents analysis to user
4. On approval: Records decision in Neo4j via `odei.neo4j.decisions.create.v1`
5. Emits: `<<<ODEI:HANDOFF:execute>>>`

**Data Passed**:

- Decision node in Neo4j (with decision ID)
- ROI breakdown
- Goal linkages
- Provenance metadata

**Execute Agent**:

1. Receives handoff signal
2. Fetches pending decision from Neo4j via `odei.neo4j.decisions.pending.v1`
3. Begins task decomposition

---

### Stage 3: Execute → Mind

**Execute Agent**:

1. Decomposes decision into tasks
2. Analyzes calendar for optimal slots
3. Proposes time blocks to user
4. On approval: Creates calendar events and records tasks in Neo4j
5. Emits: `<<<ODEI:HANDOFF:mind>>>`

**Data Passed**:

- Task nodes in Neo4j (linked to decision)
- Calendar event references
- Execution plan metadata

**Mind Agent**:

1. Receives handoff signal
2. Adds execution data to analysis pipeline
3. Runs pattern detection on completed work

---

### Stage 4: Mind → Feedback Loop

**Mind Agent**:

1. Analyzes patterns across all historical data
2. Identifies blind spots and optimizations
3. Records insights in Neo4j via `odei.neo4j.insights.create.v1`
4. Feeds optimizations back to all stages

**Data Passed**:

- Insight nodes in Neo4j
- Updated system parameters (ROI weights, time estimates)
- Pattern discoveries

**Feedback Loop**:

- Insights inform future Discuss validations
- Optimizations improve Decisions ROI calculations
- Patterns enhance Execute scheduling
- Continuous improvement cycle

## Data Flow Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                        User Input                            │
└──────────────────────┬──────────────────────────────────────┘
                       ↓
         ┌─────────────────────────┐
         │   Discuss Agent         │
         │   (Constitutional)      │
         └─────────┬───────────────┘
                   ↓ thread_id (odei-history)
         ┌─────────────────────────┐
         │   Decisions Agent       │
         │   (ROI Calculation)     │
         └─────────┬───────────────┘
                   ↓ decision_id (Neo4j)
         ┌─────────────────────────┐
         │   Execute Agent         │
         │   (Task + Calendar)     │
         └─────────┬───────────────┘
                   ↓ task_ids (Neo4j) + events (Apple Calendar)
         ┌─────────────────────────┐
         │   Mind Agent           │
         │   (Pattern Analysis)    │
         └─────────┬───────────────┘
                   ↓ insights (Neo4j)
              ┌────┴──────────────┐
              │  Feedback Loop     │
              │  (All Agents)      │
              └────────────────────┘
```

## MCP Data Storage

### odei-history (SQLite)

**Raw conversations**: All discussions stored as threads with messages.

```sql
threads (id, module, title, created_at, meta)
messages (id, thread_id, seq, role, body, created_at)
```

**Used by**: All agents for conversation recording

---

### odei-neo4j (Constitutional Graph)

**Canonical entities**: Only curated, validated data.

```cypher
// Discussions (curated summaries, not raw chat)
(:Discussion {id, topic, alignment, recommendation})

// Decisions (approved choices with ROI)
(:Decision {id, option, roi, reasoning, servesGoalType})

// Tasks (actionable work items)
(:Task {id, title, effortHours, priority, status})

// Insights (learned patterns)
(:Insight {id, pattern, confidence, evidence})

// Goals (multi-horizon objectives)
(:Goal {id, title, horizon, target})
```

**Used by**:

- Discuss: Reads goals, writes discussion summaries
- Decisions: Writes decisions, reads goals/patterns
- Execute: Reads decisions, writes tasks
- Mind: Reads everything, writes insights

---

### odei-apple (Apple Calendar via EventKit)

**Time blocks**: Calendar events linked to tasks.

```
Calendar Events (title, start, end, notes)
```

**Used by**:

- Discuss: Reads for workload assessment
- Execute: Reads + writes for time blocking

## Electron App Integration

### Terminal Manager

The Electron app's `agent-manager.js` listens for control frames in PTY output:

```javascript
pty.onData((data) => {
  // Scan for control frames
  const readyMatch = data.match(/<<<ODEI:READY:(.*?)>>>/);
  if (readyMatch) {
    const { stage, ts } = JSON.parse(readyMatch[1]);
    emit('agent-ready', { stage, ts });
  }

  const handoffMatch = data.match(/<<<ODEI:HANDOFF:(\w+)>>>/);
  if (handoffMatch) {
    const targetStage = handoffMatch[1];
    emit('agent-handoff', { from: currentStage, to: targetStage });
    // Switch active terminal tab to target stage
  }
});
```

### User Actions

- User initiates workflow in **Discuss** terminal
- Discuss validates → emits `<<<ODEI:HANDOFF:decisions>>>`
- Electron app auto-switches to **Decisions** terminal tab
- Decisions calculates ROI → emits `<<<ODEI:HANDOFF:execute>>>`
- Electron app auto-switches to **Execute** terminal tab
- Execute creates plan → emits `<<<ODEI:HANDOFF:mind>>>`
- Mind runs in background, surfaces insights periodically

## Error Handling

### Agent Crash

If an agent process dies:

1. Electron app detects PTY exit
2. Logs error with context
3. Shows user notification
4. Offers restart with exponential backoff

### MCP Connection Failure

If MCP server is unreachable:

1. Agent emits: `<<<ODEI:ERROR:{"code":"MCP_CONNECTION_FAILED"}>>>`
2. Electron app shows error in UI
3. User can check MCP server health in status panel

### Invalid Handoff

If stage emits handoff to non-existent stage:

1. Electron app logs warning
2. Stays on current stage
3. Does not auto-switch

## Best Practices

### For Agent Prompts

- ✅ Always emit READY frame on initialization
- ✅ Always emit HANDOFF frame when passing work
- ✅ Always record intermediate data in MCP servers
- ✅ Never skip stages (e.g., Discuss → Execute directly)

### For Electron App

- ✅ Parse control frames reliably (regex with capture groups)
- ✅ Handle malformed frames gracefully (log and ignore)
- ✅ Provide manual stage switching (don't force auto-switch)
- ✅ Show agent status in UI (ready, working, crashed)

### For Users

- ✅ Let workflow complete naturally (don't interrupt handoffs)
- ✅ Review each stage's output before approving
- ✅ Use Mind agent insights to improve future decisions

## Testing Control Frames

### Manual Test

```bash
cd /Users/ai/ODEI/agents/discuss
claude --interactive --dangerously-skip-permissions

# Agent should emit:
# <<<ODEI:READY:{"stage":"discuss","ts":"..."}>>>
```

### Automated Test

```javascript
// In Electron app tests
const pty = spawn('claude', ['--interactive'], { cwd: '/Users/ai/ODEI/agents/discuss' });

const frames = [];
pty.onData((data) => {
  const matches = data.matchAll(/<<<ODEI:(\w+):(.*?)>>>/g);
  for (const match of matches) {
    frames.push({ type: match[1], payload: match[2] });
  }
});

// Assert READY frame appears within 5 seconds
await waitFor(() => frames.some((f) => f.type === 'READY'));
```

## Future Enhancements

- **Handoff Metadata**: Include more context in handoff payload (summary, key decisions)
- **Branching Workflows**: Support conditional handoffs (e.g., Execute → Mind OR Discuss if blocked)
- **Parallel Stages**: Run Discuss + Mind simultaneously for proactive suggestions
- **Handoff History**: Log all handoffs in Neo4j for workflow analysis

---

**Key Principle**: Control frames provide lightweight, text-based coordination without requiring complex IPC or state management. Agents remain independent, and the Electron app acts as an orchestrator.
