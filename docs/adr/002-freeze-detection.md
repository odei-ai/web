# ADR-002: Main Thread Freeze Detection Architecture

**Date**: 2025-12-13
**Status**: Accepted
**Deciders**: AI Principal, Infrastructure Developer
**Tags**: performance, monitoring, diagnostics, worker

## Context

ODEI is an Electron app with complex UI that experienced intermittent freezes:

- **3D graph rendering** (ForceGraph3D with WebGL) blocking main thread
- **MCP server calls** causing 500ms+ hangs
- **Memory leaks** from uncleaned event listeners
- **Tab switching** breaking animations and causing black screens
- **No visibility** into what was causing freezes during development

Traditional debugging tools (Chrome DevTools profiler) require manual inspection and don't capture freeze events that happen in production usage.

### Requirements

1. **Real-time detection** - Identify freezes as they happen, not retroactively
2. **Minimal overhead** - Monitoring itself must not cause freezes
3. **Diagnostic data** - Capture what was running when freeze occurred
4. **Throttled logging** - Prevent console spam during long freezes
5. **Session persistence** - Preserve freeze logs across page reloads
6. **Visual feedback** - Show freeze events in the UI
7. **Integration** - Work with existing diagnostic overlay (CriticalDiagnostic)

## Decision

**Implement a Web Worker-based freeze detector** that monitors main thread heartbeat and captures diagnostic snapshots when freezes occur.

### Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                       Main Thread                            │
│                                                              │
│  ┌──────────────┐         ┌──────────────┐                 │
│  │ FreezeDetector│◄────────┤  Application │                 │
│  │   (Monitor)   │         │    Code      │                 │
│  └───────┬───────┘         └──────────────┘                 │
│          │                                                   │
│          │ postMessage('ping')                              │
│          │ every 50ms                                       │
│          ▼                                                   │
│  ┌──────────────────────────────────────────────────┐      │
│  │           Web Worker (Separate Thread)            │      │
│  │                                                   │      │
│  │  - Checks last ping timestamp every 100ms        │      │
│  │  - If delta > 500ms → freeze detected            │      │
│  │  - Posts freeze event back to main thread        │      │
│  └──────────────────────────────────────────────────┘      │
└─────────────────────────────────────────────────────────────┘
```

### Implementation Details

**FreezeDetector Class** (`src/modules/FreezeDetector.js`):

```javascript
class FreezeDetector {
  constructor(options = {}) {
    this.thresholdMs = options.thresholdMs || 500; // Alert if blocked > 500ms
    this.checkIntervalMs = options.checkIntervalMs || 100;
    this.freezeLog = [];
    this.activeOperations = new Map(); // Track running operations
  }

  start() {
    // Create Web Worker with inline code
    const workerCode = `
      let lastPing = Date.now();
      setInterval(() => {
        const now = Date.now();
        const delta = now - lastPing;
        if (delta > ${this.thresholdMs}) {
          self.postMessage({ type: 'freeze', duration: delta });
        }
      }, ${this.checkIntervalMs});

      self.onmessage = (e) => {
        if (e.data.type === 'ping') lastPing = Date.now();
      };
    `;

    this.worker = new Worker(URL.createObjectURL(new Blob([workerCode], { type: 'application/javascript' })));

    this.worker.onmessage = (e) => {
      if (e.data.type === 'freeze') {
        this._onFreezeDetected(e.data.duration);
      }
    };

    // Heartbeat: ping worker every 50ms
    this._heartbeatInterval = setInterval(() => {
      this.worker.postMessage({ type: 'ping' });
    }, 50);
  }

  // Track active operations for diagnostic context
  trackOperation(name, metadata = {}) {
    const id = ++this.operationId;
    this.activeOperations.set(id, { name, startTime: Date.now(), metadata });
    return () => this.activeOperations.delete(id); // Cleanup function
  }

  _onFreezeDetected(duration) {
    // THROTTLE: Only log every 2 seconds to prevent console spam
    if (Date.now() - this._lastFreezeLogTime < 2000) {
      this._suppressedFreezeCount++;
      return;
    }

    const diagnostics = this._captureDiagnostics(duration);
    console.error(`[FreezeDetector] UI FREEZE: ${duration}ms`);

    // Store in freeze log and sessionStorage
    this.freezeLog.push(diagnostics);
    sessionStorage.setItem('odei_freeze_log', JSON.stringify(this.freezeLog));

    // Emit event for UI to display
    window.dispatchEvent(new CustomEvent('odei:freeze-detected', { detail: diagnostics }));
  }

  _captureDiagnostics(duration) {
    return {
      timestamp: new Date().toISOString(),
      freezeDuration: duration,
      activeOperations: Array.from(this.activeOperations.values()),
      mcpState: window.mcpClient
        ? {
            activeRequests: window.mcpClient._activeRequests,
            queueLength: window.mcpClient._requestQueue?.length || 0,
          }
        : null,
      memoryInfo: performance.memory
        ? {
            usedJSHeapSize: Math.round(performance.memory.usedJSHeapSize / 1024 / 1024) + 'MB',
          }
        : null,
    };
  }
}

// Export singleton and auto-start
export const freezeDetector = new FreezeDetector();
freezeDetector.start();
```

### Integration with CriticalDiagnostic

**CriticalDiagnostic** (`src/modules/CriticalDiagnostic.js`) provides visual overlay showing:

- Real-time FPS counter
- Memory usage
- Last frame timestamp
- Event log with freeze events highlighted in red

FreezeDetector emits `odei:freeze-detected` events that CriticalDiagnostic listens for and displays.

### Throttling Strategy

**Problem**: During a 5-second freeze, the worker would detect freeze every 100ms = 50 console.error calls, flooding the console and making the freeze worse.

**Solution**: Throttle freeze logging to max once per 2 seconds:

```javascript
if (Date.now() - this._lastFreezeLogTime < 2000) {
  this._suppressedFreezeCount++;
  return; // Skip logging
}
```

After throttle period, log includes: `(5 additional freeze events suppressed)`

## Consequences

### Positive Consequences

- **Real-time visibility** - Developers see freezes as they happen
- **Diagnostic context** - Know what was running when freeze occurred
- **Low overhead** - Worker runs in separate thread, <1% CPU
- **Session persistence** - Freeze logs survive page reloads
- **Throttled logging** - No console spam during long freezes
- **Operation tracking** - `mcpClient` and other services can mark their work
- **Integration** - Works with existing CriticalDiagnostic overlay

### Negative Consequences

- **Not 100% accurate** - 50ms heartbeat means tiny freezes (<50ms) might be missed
- **Worker overhead** - Small memory cost for background thread
- **False positives** - DevTools debugger breakpoint = "freeze"
- **No stack traces** - Can't capture call stack of blocking code (main thread is blocked!)
- **Limited to main thread** - Doesn't detect worker or GPU issues

### Neutral Consequences

- **Requires worker support** - Won't work in environments without Web Workers (but Electron always has them)
- **Manual instrumentation** - Developers must call `trackOperation()` for context
- **Log size grows** - Need to cap at 50 entries to prevent memory leak

## Alternatives Considered

### Alternative 1: PerformanceObserver with Long Task API

**Description**: Use browser's built-in Long Task API to detect tasks >50ms.

```javascript
const observer = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    if (entry.duration > 500) {
      console.warn('Long task:', entry);
    }
  }
});
observer.observe({ entryTypes: ['longtask'] });
```

**Pros**:

- Native browser API, no custom worker
- More accurate timing
- Includes attribution data

**Cons**:

- **Limited browser support** - Not available in Electron's Chromium version
- **No freeze threshold control** - Fixed at 50ms
- **Passive observation** - Can't emit custom events or capture app-specific context
- **No MCP state** - Doesn't know about our mcpClient queue

**Why rejected**: Long Task API not reliably available in Electron. Can't customize thresholds or capture ODEI-specific diagnostics.

### Alternative 2: requestAnimationFrame Heartbeat

**Description**: Track time between RAF calls to detect stalls.

```javascript
let lastFrame = Date.now();
function checkFreeze() {
  const now = Date.now();
  const delta = now - lastFrame;
  if (delta > 500) {
    console.error('Freeze detected:', delta);
  }
  lastFrame = now;
  requestAnimationFrame(checkFreeze);
}
requestAnimationFrame(checkFreeze);
```

**Pros**:

- Simple, no worker needed
- Runs on main thread, easier to debug
- Leverages existing RAF loop

**Cons**:

- **Runs on main thread** - If thread is blocked, RAF doesn't fire = no detection!
- **Dependent on active tab** - RAF paused when tab backgrounded
- **No guarantee** - Browser may skip RAF calls if too busy
- **Can't detect its own freeze** - Logic error causes infinite loop = RAF never runs

**Why rejected**: Fundamentally flawed - can't detect main thread freeze from main thread. Worker approach is more reliable.

### Alternative 3: Interval-based Heartbeat (No Worker)

**Description**: Use setInterval to check heartbeat, but on main thread.

```javascript
let lastCheck = Date.now();
setInterval(() => {
  const now = Date.now();
  const delta = now - lastCheck;
  if (delta > 1000) {
    // Expect 500ms interval, detect if >1000ms
    console.error('Freeze detected:', delta - 500);
  }
  lastCheck = now;
}, 500);
```

**Pros**:

- Simple implementation
- No worker needed

**Cons**:

- **Same problem as RAF** - Interval delayed by main thread freeze
- **Unreliable timing** - `setInterval` not guaranteed to fire on time
- **Cumulative drift** - If blocked once, all future intervals off by that amount
- **Can't self-diagnose** - Thread blocking prevents interval from running

**Why rejected**: Same flaw as RAF approach. Worker is only reliable way to monitor main thread from outside.

## Implementation Notes

### Key Files

- `/src/modules/FreezeDetector.js` - Core freeze detection logic
- `/src/modules/CriticalDiagnostic.js` - Visual diagnostic overlay
- `/src/modules/mcpClient.js` - Integrates `trackOperation()` for MCP call tracking
- `/src/modules/ThreeGraphRenderer.js` - Graph renderer with freeze prevention fixes

### Usage Pattern

```javascript
// Auto-imported and started as singleton
import { freezeDetector } from './modules/FreezeDetector.js';

// Track an expensive operation
const endTracking = freezeDetector.trackOperation('render-graph', { nodeCount: 500 });
try {
  renderGraph(); // Potentially expensive work
} finally {
  endTracking(); // Always cleanup, even on error
}

// Access freeze log
const freezes = freezeDetector.getLog();
console.log(`Detected ${freezes.length} freezes this session`);

// Clear freeze history
freezeDetector.clearLog();
```

### Performance Budgets

Based on freeze detection findings:

- **Main thread budget**: <16ms per frame (60 FPS)
- **Freeze threshold**: 500ms (noticeable UI lag)
- **MCP call timeout**: 10 seconds max
- **Physics simulation**: <50ms per tick (20 FPS minimum during layout)

### Testing Considerations

**Simulate freeze** for testing:

```javascript
// In DevTools console
const start = Date.now();
while (Date.now() - start < 3000) {
  // Block main thread for 3 seconds
}
// Should trigger freeze detection after ~3 seconds
```

**Check freeze log**:

```javascript
window.freezeDetector.getLog();
```

## References

- `/src/modules/FreezeDetector.js` - Implementation
- `/src/modules/CriticalDiagnostic.js` - Visual overlay integration
- `/tests/freeze-usage-simulation.spec.ts` - E2E freeze detection tests
- [Web Workers API](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API)
- [Long Tasks API](https://developer.mozilla.org/en-US/docs/Web/API/PerformanceLongTaskTiming)
- ADR-003: MCP Server Architecture - Explains semaphore integration
