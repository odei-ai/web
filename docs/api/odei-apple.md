# odei-apple API Reference

Swift-based MCP server for Apple Health and HealthKit integration.

## Overview

The odei-apple server provides access to Apple Health data from HealthKit, including vitals, workouts, sleep tracking, heart rate variability (HRV), and environmental data.

**Platform:** macOS only (requires HealthKit access)  
**Language:** Swift  
**Build:** `swift build -c release`

## Table of Contents

- [odei.apple.health.v1](#odeiapplehealthv1)
- [odei.apple.health.vitals.v1](#odeiapplehealthvitalsv1)
- [odei.apple.health.hrv.v1](#odeiapplehealthhrvv1)
- [odei.apple.health.workouts.v1](#odeiapplehealthworkoutsv1)
- [odei.apple.health.sleep.v1](#odeiapplehealthsleepv1)
- [odei.apple.health.environment.v1](#odeiapplehealthenvironmentv1)
- [odei.apple.health.activity.v1](#odeiapplehealthactivityv1)
- [odei.apple.health.heartRate.v1](#odeiapplehealthheartratev1)
- [odei.apple.health.readiness.v1](#odeiapplehealthreadinessv1)
- [odei.apple.health.authorize.v1](#odeiapplehealthauthorizev1)

---

## odei.apple.health.v1

Get comprehensive health summary including all available metrics.

### Parameters

None

### Returns

```typescript
{
  ok: boolean;
  timestamp: string;
  vitals: {
    heartRate: number;
    hrvSDNN: number;
    respiratoryRate: number;
    vo2Max: number;
    bodyTemperature: number;
  };
  activity: {
    steps: number;
    activeEnergy: number;
    exerciseMinutes: number;
    standHours: number;
  };
  sleep: {
    duration: number;
    quality: string;
    stages: {
      deep: number;
      rem: number;
      core: number;
      awake: number;
    };
  };
}
```

### Example

```typescript
// Get comprehensive health data
const result = await mcp.callTool('odei.apple.health.v1', {});
console.log(\`Heart Rate: \${result.vitals.heartRate} bpm\`);
```

---

## Setup

### 1. Build the Server

```bash
cd servers/odei-apple
swift build -c release
```

### 2. Configure MCP

Add to `.claude/mcp.json`:

```json
{
  "mcpServers": {
    "odei-apple": {
      "command": "/Users/ai/ODEI/servers/odei-apple/.build/release/odei-apple"
    }
  }
}
```

### 3. Grant Permissions

On first run, macOS will prompt for HealthKit access. Grant all requested permissions.

## Privacy & Security

- All data stays local on the Mac
- No network requests
- Respects HealthKit privacy model
- Requires explicit user authorization
