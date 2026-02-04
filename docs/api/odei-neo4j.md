# odei-neo4j API Reference

Complete API documentation for the odei-neo4j MCP server.

## Table of Contents

- [odei.neo4j.backup.restore.v1](#odeineo4jbackuprestorev1)
- [odei.neo4j.cache.management.v1](#odeineo4jcachemanagementv1)
- [odei.neo4j.context.add.v1](#odeineo4jcontextaddv1)
- [odei.neo4j.context.usage.v1](#odeineo4jcontextusagev1)
- [odei.neo4j.discuss.activeThreads.v1](#odeineo4jdiscussactivethreadsv1)
- [odei.neo4j.discuss.stats.v1](#odeineo4jdiscussstatsv1)
- [odei.neo4j.execute.tasksToday.v1](#odeineo4jexecutetaskstodayv1)
- [odei.neo4j.execute.workSessions.v1](#odeineo4jexecuteworksessionsv1)
- [odei.neo4j.foundation.list.v2](#odeineo4jfoundationlistv2)
- [odei.neo4j.graph.getAll.v1](#odeineo4jgraphgetallv1)
- [odei.neo4j.hybrid.plusplus.search.v1](#odeineo4jhybridplusplussearchv1)
- [odei.neo4j.hybrid.search.v1](#odeineo4jhybridsearchv1)
- [odei.neo4j.memory.coverage.v1](#odeineo4jmemorycoveragev1)
- [odei.neo4j.memory.retrieve.v1](#odeineo4jmemoryretrievev1)
- [odei.neo4j.mind.patternStats.v1](#odeineo4jmindpatternstatsv1)
- [odei.neo4j.mind.recentInsights.v1](#odeineo4jmindrecentinsightsv1)
- [odei.neo4j.node.update.v1](#odeineo4jnodeupdatev1)
- [odei.neo4j.personalized.search.v1](#odeineo4jpersonalizedsearchv1)
- [odei.neo4j.plan.backlog.v1](#odeineo4jplanbacklogv1)
- [odei.neo4j.plan.metrics.v1](#odeineo4jplanmetricsv1)
- [odei.neo4j.preflight.validate.v1](#odeineo4jpreflightvalidatev1)
- [odei.neo4j.relationship.create.v1](#odeineo4jrelationshipcreatev1)
- [odei.neo4j.schema.inventory.v1](#odeineo4jschemainventoryv1)
- [odei.neo4j.temporal.autofix.v1](#odeineo4jtemporalautofixv1)
- [odei.neo4j.temporal.context.v1](#odeineo4jtemporalcontextv1)
- [odei.neo4j.user.context.track.v1](#odeineo4jusercontexttrackv1)
- [odei.neo4j.vision.list.v2](#odeineo4jvisionlistv2)
- [odei.neo4j.workload.assess.v1](#odeineo4jworkloadassessv1)

---

## odei.neo4j.backup.restore.v1

No description provided

### Parameters

| Name | Type | Description |
|------|------|-------------|
| `key` | string (optional>.nullable(> | |
| `labels` | array<z.string.min(1>> | |
| `properties` | record(z.any(>> | |

### Example

```typescript
// Call odei.neo4j.backup.restore.v1
const result = await mcp.callTool('odei.neo4j.backup.restore.v1', {
  labels: /* array<z.string.min(1>> */,
  properties: /* record(z.any(>> */,
});
```

---

## odei.neo4j.cache.management.v1

No description provided

### Example

```typescript
// Call odei.neo4j.cache.management.v1
const result = await mcp.callTool('odei.neo4j.cache.management.v1', {
});
```

---

## odei.neo4j.context.add.v1

No description provided

### Parameters

| Name | Type | Description |
|------|------|-------------|
| `layer` | enum(['system' | |
| `label` | string.describe('Human-readable label for this context item (e.g. | |
| `tokens` | number.int(>.positive(>.describe('Estimated token count for this item'> | |
| `content` | string (optional>.describe('Optional preview of the content being added'> | |
| `timestamp` | string (optional>.describe('Optional ISO timestamp (defaults to now>'> | |

### Example

```typescript
// Call odei.neo4j.context.add.v1
const result = await mcp.callTool('odei.neo4j.context.add.v1', {
  layer: /* enum(['system' */,
  label: /* string.describe('Human-readable label for this context item (e.g. */,
  tokens: /* number.int(>.positive(>.describe('Estimated token count for this item'> */,
});
```

---

## odei.neo4j.context.usage.v1

No description provided

### Parameters

| Name | Type | Description |
|------|------|-------------|
| `module` | enum(['discuss' | |

### Example

```typescript
// Call odei.neo4j.context.usage.v1
const result = await mcp.callTool('odei.neo4j.context.usage.v1', {
  module: /* enum(['discuss' */,
});
```

---

## odei.neo4j.discuss.activeThreads.v1

No description provided

### Parameters

| Name | Type | Description |
|------|------|-------------|
| `limit` | number.int(>.positive(>.max(100> (optional> | |

### Example

```typescript
// Call odei.neo4j.discuss.activeThreads.v1
const result = await mcp.callTool('odei.neo4j.discuss.activeThreads.v1', {
});
```

---

## odei.neo4j.discuss.stats.v1

No description provided

### Parameters

| Name | Type | Description |
|------|------|-------------|
| `limit` | number.int(>.positive(>.max(100> (optional> | |

### Example

```typescript
// Call odei.neo4j.discuss.stats.v1
const result = await mcp.callTool('odei.neo4j.discuss.stats.v1', {
});
```

---

## odei.neo4j.execute.tasksToday.v1

No description provided

### Example

```typescript
// Call odei.neo4j.execute.tasksToday.v1
const result = await mcp.callTool('odei.neo4j.execute.tasksToday.v1', {
});
```

---

## odei.neo4j.execute.workSessions.v1

No description provided

### Example

```typescript
// Call odei.neo4j.execute.workSessions.v1
const result = await mcp.callTool('odei.neo4j.execute.workSessions.v1', {
});
```

---

## odei.neo4j.foundation.list.v2

No description provided

### Example

```typescript
// Call odei.neo4j.foundation.list.v2
const result = await mcp.callTool('odei.neo4j.foundation.list.v2', {
});
```

---

## odei.neo4j.graph.getAll.v1

No description provided

### Example

```typescript
// Call odei.neo4j.graph.getAll.v1
const result = await mcp.callTool('odei.neo4j.graph.getAll.v1', {
});
```

---

## odei.neo4j.hybrid.plusplus.search.v1

No description provided

### Example

```typescript
// Call odei.neo4j.hybrid.plusplus.search.v1
const result = await mcp.callTool('odei.neo4j.hybrid.plusplus.search.v1', {
});
```

---

## odei.neo4j.hybrid.search.v1

No description provided

### Example

```typescript
// Call odei.neo4j.hybrid.search.v1
const result = await mcp.callTool('odei.neo4j.hybrid.search.v1', {
});
```

---

## odei.neo4j.memory.coverage.v1

No description provided

### Parameters

| Name | Type | Description |
|------|------|-------------|
| `layers` | array<z.enum(['foundation' | |

### Example

```typescript
// Call odei.neo4j.memory.coverage.v1
const result = await mcp.callTool('odei.neo4j.memory.coverage.v1', {
  layers: /* array<z.enum(['foundation' */,
});
```

---

## odei.neo4j.memory.retrieve.v1

Constitutional decisions, life direction, goal alignment, workload risk

### Example

```typescript
// Call odei.neo4j.memory.retrieve.v1
const result = await mcp.callTool('odei.neo4j.memory.retrieve.v1', {
});
```

---

## odei.neo4j.mind.patternStats.v1

No description provided

### Example

```typescript
// Call odei.neo4j.mind.patternStats.v1
const result = await mcp.callTool('odei.neo4j.mind.patternStats.v1', {
});
```

---

## odei.neo4j.mind.recentInsights.v1

No description provided

### Parameters

| Name | Type | Description |
|------|------|-------------|
| `limit` | number.int(>.positive(>.max(10> (optional> | |

### Example

```typescript
// Call odei.neo4j.mind.recentInsights.v1
const result = await mcp.callTool('odei.neo4j.mind.recentInsights.v1', {
});
```

---

## odei.neo4j.node.update.v1

No description provided

### Example

```typescript
// Call odei.neo4j.node.update.v1
const result = await mcp.callTool('odei.neo4j.node.update.v1', {
});
```

---

## odei.neo4j.personalized.search.v1

No description provided

### Example

```typescript
// Call odei.neo4j.personalized.search.v1
const result = await mcp.callTool('odei.neo4j.personalized.search.v1', {
});
```

---

## odei.neo4j.plan.backlog.v1

No description provided

### Parameters

| Name | Type | Description |
|------|------|-------------|
| `limit` | number.int(>.positive(>.max(100> (optional> | |

### Example

```typescript
// Call odei.neo4j.plan.backlog.v1
const result = await mcp.callTool('odei.neo4j.plan.backlog.v1', {
});
```

---

## odei.neo4j.plan.metrics.v1

No description provided

### Example

```typescript
// Call odei.neo4j.plan.metrics.v1
const result = await mcp.callTool('odei.neo4j.plan.metrics.v1', {
});
```

---

## odei.neo4j.preflight.validate.v1

No description provided

### Parameters

| Name | Type | Description |
|------|------|-------------|
| `type` | string.describe('Relationship type (e.g. | |
| `targetId` | string.describe('Target node ID'> | |
| `direction` | enum(['outgoing' | |

### Example

```typescript
// Call odei.neo4j.preflight.validate.v1
const result = await mcp.callTool('odei.neo4j.preflight.validate.v1', {
  type: /* string.describe('Relationship type (e.g. */,
  targetId: /* string.describe('Target node ID'> */,
  direction: /* enum(['outgoing' */,
});
```

---

## odei.neo4j.relationship.create.v1

No description provided

### Example

```typescript
// Call odei.neo4j.relationship.create.v1
const result = await mcp.callTool('odei.neo4j.relationship.create.v1', {
});
```

---

## odei.neo4j.schema.inventory.v1

No description provided

### Example

```typescript
// Call odei.neo4j.schema.inventory.v1
const result = await mcp.callTool('odei.neo4j.schema.inventory.v1', {
});
```

---

## odei.neo4j.temporal.autofix.v1

Goals with passed deadlines that were never marked as completed

### Parameters

| Name | Type | Description |
|------|------|-------------|
| `issueType` | enum(['STALE_GOAL' | |
| `dryRun` | boolean (optional> | |

### Example

```typescript
// Call odei.neo4j.temporal.autofix.v1
const result = await mcp.callTool('odei.neo4j.temporal.autofix.v1', {
  issueType: /* enum(['STALE_GOAL' */,
});
```

---

## odei.neo4j.temporal.context.v1

No description provided

### Parameters

| Name | Type | Description |
|------|------|-------------|
| `timezone` | string (optional> | |

### Example

```typescript
// Call odei.neo4j.temporal.context.v1
const result = await mcp.callTool('odei.neo4j.temporal.context.v1', {
});
```

---

## odei.neo4j.user.context.track.v1

No description provided

### Example

```typescript
// Call odei.neo4j.user.context.track.v1
const result = await mcp.callTool('odei.neo4j.user.context.track.v1', {
});
```

---

## odei.neo4j.vision.list.v2

No description provided

### Example

```typescript
// Call odei.neo4j.vision.list.v2
const result = await mcp.callTool('odei.neo4j.vision.list.v2', {
});
```

---

## odei.neo4j.workload.assess.v1

No description provided

### Example

```typescript
// Call odei.neo4j.workload.assess.v1
const result = await mcp.callTool('odei.neo4j.workload.assess.v1', {
});
```

---
