# Sovereign Writes Architecture

## Overview

ODEI implements a **Sovereign Writes** architecture where all graph mutations flow through a controlled pipeline with Guardian validation. Direct writes to Neo4j from the frontend are forbidden.

## Why Sovereign Writes?

Direct Neo4j writes bypass critical safety measures:
- No time window checks (Eva Time, Evening Critic, Focus Block)
- No layer protection (Foundation/Vision require human approval)
- No audit trail (Evidence log)
- No approval workflow
- No constitutional alignment checks

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    FRONTEND                                  │
│  TodayView, DetailPanel, MemoryViewManager                  │
│                         │                                    │
│                         ▼                                    │
│            WorldModelWriteGateway.js                        │
│            (SINGLE entry point for writes)                  │
└─────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│               odei-worldmodel (MCP Server)                  │
│                                                              │
│  Proposal Pipeline:                                          │
│  1. create_proposal - Creates proposal with GraphOps        │
│  2. decide_proposal - APPROVED or REJECTED                  │
│  3. apply_proposal  - Executes GraphOps (Guardian checks)   │
│  4. record_outcome  - Records verification                   │
│                         │                                    │
│         ┌───────────────┴───────────────┐                   │
│         │      UNIFIED GUARDIAN         │                   │
│         │                               │                   │
│         │  Reads Settings UI config:    │                   │
│         │  - Time windows (Eva Time,    │                   │
│         │    Evening Critic, Focus)     │                   │
│         │  - Layer protection           │                   │
│         │  - Approval thresholds        │                   │
│         │  - Calendar budgets           │                   │
│         │  - Human-only entities        │                   │
│         └───────────────────────────────┘                   │
│                         │                                    │
│                         ▼                                    │
│         ┌───────────────────────────────┐                   │
│         │     Evidence (SQLite)         │                   │
│         │  Append-only audit log        │                   │
│         └───────────────────────────────┘                   │
└─────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│                    odei-neo4j                                │
│                                                              │
│  READS ONLY from frontend:                                   │
│  - odei.neo4j.graph.getAll.v1                               │
│  - odei.neo4j.graph.getById.v1                              │
│  - odei.neo4j.graph.search.v1                               │
│                                                              │
│  WRITES blocked by preload guard                             │
└─────────────────────────────────────────────────────────────┘
```

## Key Components

### 1. WorldModelWriteGateway (`src/modules/worldmodel/WorldModelWriteGateway.js`)

The **single entry point** for all frontend writes. Provides:

```javascript
// Create/Update/Delete nodes
worldModelGateway.createNode(labels, properties, meta)
worldModelGateway.updateNode(entityId, patch, meta)
worldModelGateway.deleteNode(entityId, meta)

// Relationships
worldModelGateway.createRelationship(fromId, relType, toId, props, meta)
worldModelGateway.deleteRelationship(relId, meta)

// Convenience methods - Tasks
worldModelGateway.createTask(taskData, meta)
worldModelGateway.updateTaskStatus(taskId, status, meta)

// Convenience methods - Scheduling
worldModelGateway.createTimeBlockIntent(intentData, meta)
worldModelGateway.createCalendarEvent(eventData, meta)

// Convenience methods - Decisions & Tracking
worldModelGateway.createProposal(proposalData, meta)
worldModelGateway.createOutcome(outcomeData, meta)
worldModelGateway.createEvidenceEntry(evidenceData, meta)

// Convenience methods - UI Operations
worldModelGateway.persistManualOrder(orderData, meta)
worldModelGateway.updateNodeViaProposal(entityId, patch, meta)
```

Meta parameter:
```javascript
{
  actor: 'human' | 'agent',
  source: 'frontend',
  rationale: 'Why this change',
  priority: 'LOW' | 'NORMAL' | 'HIGH',
  autoApprove: boolean
}
```

### 2. Guardian Config Integration (`servers/odei-worldmodel/src/config/guardianConfigLoader.ts`)

Loads the Settings UI configuration from:
```
~/Library/Application Support/odei/guardian-config.json
```

Enforced checks:
| Check | Code | Description |
|-------|------|-------------|
| Eva Time | `EVA_TIME_BLOCKED` | Family time protection |
| Focus Block | `FOCUS_BLOCK_ACTIVE` | Deep work protection |
| Weekend Cooldown | `WEEKEND_COOLDOWN` | Weekend quiet mode |
| Evening Critic | `EVENING_CRITIC_BLOCKED` | Foundation/Vision changes after 20:00 |
| Layer Protection | `LAYER_REQUIRES_HUMAN` | Per-layer approval requirements |
| Delete Protection | `DELETE_REQUIRES_HUMAN` | Human approval for deletions |

### 3. Runtime Write Guard (`app-main/preload.js`)

Defense-in-depth: blocks any direct Neo4j writes from frontend.

```javascript
const FORBIDDEN_NEO4J_WRITES = [
  'odei.neo4j.node.create',
  'odei.neo4j.node.update',
  'odei.neo4j.node.delete',
  'odei.neo4j.task.create',
  'odei.neo4j.task.update',
  'odei.neo4j.execution.create',
  'odei.neo4j.track.create',
  'odei.neo4j.mind.create',
  'odei.neo4j.relationship.create',
  'odei.neo4j.relationship.delete',
];
```

If any code attempts a forbidden write:
```
[SECURITY] Blocked direct Neo4j write: odei.neo4j.node.update.v1
Error: Direct Neo4j writes are forbidden. Use odei-worldmodel pipeline.
```

## Refactored Components

### CommandDataService.js

All 11 write methods now route through WorldModelWriteGateway:
- `createTask()` → `worldModelGateway.createTask()`
- `updateTaskStatus()` → `worldModelGateway.updateTaskStatus()`
- `updateNodeStatus()` → `worldModelGateway.updateNode()`
- `updateNodeFields()` → `worldModelGateway.updateNode()`
- `createTimeBlockIntent()` → `worldModelGateway.createTimeBlockIntent()`
- `createProposal()` → `worldModelGateway.createProposal()`
- `createCalendarEvent()` → `worldModelGateway.createCalendarEvent()`
- `createOutcome()` → `worldModelGateway.createOutcome()`
- `createEvidenceEntry()` → `worldModelGateway.createEvidenceEntry()`
- `createRelationship()` → `worldModelGateway.createRelationship()`
- `persistManualOrder()` → `worldModelGateway.persistManualOrder()`

### MemoryViewManager.js

- **Delete operation**: Now uses `worldModelGateway.deleteNode()` with HIGH priority
- **Edit operation**: Now uses `worldModelGateway.updateNodeViaProposal()` with full workflow

## Testing

To verify the guard works:
```javascript
// This should throw an error:
await window.odei.data.get('odei-neo4j', 'odei.neo4j.node.update.v1', { id: 'test' });
// Expected: Error "Direct Neo4j writes are forbidden..."
```

## Migration Notes

If you have code that calls Neo4j directly:

```javascript
// BEFORE (will be blocked):
await callMCPTool('odei-neo4j', 'odei.neo4j.node.update.v1', { id, data });

// AFTER (correct):
import { worldModelGateway } from './worldmodel/WorldModelWriteGateway.js';
await worldModelGateway.updateNode(id, data, { actor: 'human', rationale: 'Update reason' });
```

## Configuration

Guardian settings are managed through the Settings UI:
- **Settings → Guardian → Time Protection**: Eva Time, Focus Block, Weekend Cooldown
- **Settings → Guardian → Decision Guardrails**: Approval thresholds
- **Settings → Guardian → Layer Protection**: Per-layer requirements

These settings are saved to `guardian-config.json` and read by odei-worldmodel at proposal validation time.
