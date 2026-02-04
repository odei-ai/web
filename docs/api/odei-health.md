# odei-health API Reference

Complete API documentation for the odei-health MCP server.

## Table of Contents

- [odei.health.readiness.v1](#odeihealthreadinessv1)
- [odei.health.state.v1](#odeihealthstatev1)

---

## odei.health.readiness.v1

Get detailed readiness analysis with factors, alerts, and recommendations.

Returns comprehensive readiness information including:
- Readiness score with contributing factors (sleep, HRV, recovery, strain)
- Active health alerts that need attention
- Personalized recommendations based on current state
- Capacity advice for planning the day

Use this tool when you need deeper insight into the user's recovery state
or when planning workload and exercise for the day.

### Example

```typescript
// Call odei.health.readiness.v1
const result = await mcp.callTool('odei.health.readiness.v1', {
});
```

---

## odei.health.state.v1

Get current health state summary from Apple Watch and Garmin data.

Returns a token-efficient summary including:
- Readiness score (0-100%) and mode (FULL/SOFT/NO-RISK)
- Key metrics: sleep hours, HRV, resting heart rate, activity
- Active alerts (low sleep, low HRV, low readiness)
- Capacity estimate for work/exercise

Use this tool to understand the user's current physical state before
making recommendations about workload, exercise, or scheduling.

### Parameters

| Name | Type | Description |
|------|------|-------------|
| `include_raw` | boolean (optional>.describe('Include raw metrics in addition to summary'> | |

### Example

```typescript
// Call odei.health.state.v1
const result = await mcp.callTool('odei.health.state.v1', {
});
```

---
