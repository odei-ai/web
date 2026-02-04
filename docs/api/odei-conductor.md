# odei-conductor API Reference

Complete API documentation for the odei-conductor MCP server.

## Table of Contents

- [odei.conductor.cancelTask.v1](#odeiconductorcanceltaskv1)
- [odei.conductor.dispatch.v1](#odeiconductordispatchv1)
- [odei.conductor.getOutput.v1](#odeiconductorgetoutputv1)
- [odei.conductor.status.v1](#odeiconductorstatusv1)
- [odei.conductor.taskComplete.v1](#odeiconductortaskcompletev1)

---

## odei.conductor.cancelTask.v1

Cancel a running task. Use this to stop a task that is taking too long or is no longer needed.

### Parameters

| Name | Type | Description |
|------|------|-------------|
| `taskId` | string.min(1> | |

### Example

```typescript
// Call odei.conductor.cancelTask.v1
const result = await mcp.callTool('odei.conductor.cancelTask.v1', {
  taskId: /* string.min(1> */,
});
```

---

## odei.conductor.dispatch.v1

Dispatch a task to another agent. Orchestration clients use this to assign work to other agents.

### Parameters

| Name | Type | Description |
|------|------|-------------|
| `agentId` | enum(['discuss' | |
| `prompt` | string.min(1>.max(100000> | |
| `timeout` | number | |
| `metadata` | record(z.unknown(>> (optional> | |

### Example

```typescript
// Call odei.conductor.dispatch.v1
const result = await mcp.callTool('odei.conductor.dispatch.v1', {
  agentId: /* enum(['discuss' */,
  prompt: /* string.min(1>.max(100000> */,
  timeout: /* number */,
});
```

---

## odei.conductor.getOutput.v1

Get output from an agent or task.

Use this to check what an agent has been doing or retrieve full logs for a completed task.

Supports two modes:
1. File-based (recommended): Provide taskId or runId to read from persisted output files
2. Real-time (legacy): Provide agentId to read from in-memory buffer

File-based mode provides complete logs with no truncation, even if real-time streaming was rate-limited.

### Example

```typescript
// Call odei.conductor.getOutput.v1
const result = await mcp.callTool('odei.conductor.getOutput.v1', {
});
```

---

## odei.conductor.status.v1

Get status of all agents and tasks. Use this to check which agents are running and their current tasks.

### Example

```typescript
// Call odei.conductor.status.v1
const result = await mcp.callTool('odei.conductor.status.v1', {
});
```

---

## odei.conductor.taskComplete.v1

Mark a task as completed. Agents call this when they finish their assigned work.

### Parameters

| Name | Type | Description |
|------|------|-------------|
| `taskId` | string.min(1> | |
| `status` | enum(['ok' | |
| `summary` | string (optional> | |
| `artifacts` | array<z.string> (optional> | |

### Example

```typescript
// Call odei.conductor.taskComplete.v1
const result = await mcp.callTool('odei.conductor.taskComplete.v1', {
  taskId: /* string.min(1> */,
  status: /* enum(['ok' */,
});
```

---
