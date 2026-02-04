# ADR-003: MCP Server Architecture and Design Principles

**Date**: 2025-12-13
**Status**: Accepted
**Deciders**: AI Principal, Infrastructure Developer
**Tags**: architecture, mcp, concurrency, performance

## Context

ODEI uses the Model Context Protocol (MCP) to communicate with external servers that provide:

- **Neo4j graph database** access (odei-neo4j)
- **Conversation history** persistence (odei-history)
- **Health data** from Apple Health and Garmin (odei-apple)
- **Gemini API** integration (odei-gemini)

Initial implementation had critical issues:

- **No concurrency control** - 50+ simultaneous MCP calls overwhelmed servers
- **Main thread blocking** - Awaiting MCP responses froze UI for 500ms+
- **No timeouts** - Hung calls blocked forever
- **No retry logic** - Transient failures crashed the app
- **Poor error handling** - Network errors propagated as uncaught rejections

### Requirements

1. **Concurrency limit** - Max 6 concurrent requests per server
2. **Timeouts** - 10 second default, configurable per tool
3. **Retry logic** - 1 automatic retry with exponential backoff
4. **Queue management** - Requests wait for available slots, with timeout
5. **Metrics** - Track latency, success rate, queue depth
6. **Integration** - Work with FreezeDetector for diagnostics

## Decision

**Implement a semaphore-based MCP client** with built-in concurrency control, timeouts, retries, and metrics.

### Architecture

```
┌───────────────────────────────────────────────────────────────┐
│                    Application Code                            │
│  (UIManager, MemoryViewManager, ConversationManager, etc.)    │
└──────────────────────┬────────────────────────────────────────┘
                       │
                       │ mcpClient.callTool()
                       ▼
┌───────────────────────────────────────────────────────────────┐
│                      MCPClient                                 │
│                   (Semaphore + Queue)                         │
│                                                               │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  Semaphore (maxConcurrent = 6)                      │    │
│  │  ┌───┬───┬───┬───┬───┬───┐                         │    │
│  │  │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │  Active slots          │    │
│  │  └───┴───┴───┴───┴───┴───┘                         │    │
│  │                                                      │    │
│  │  Queue (FIFO)                                       │    │
│  │  [req7] → [req8] → [req9] → ...                   │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                               │
│  Timeout (10s per request)                                   │
│  Retry (1 retry with 400ms backoff)                         │
│  Metrics (latency, success rate, queue depth)                │
└───────────────────────┬───────────────────────────────────────┘
                        │
                        │ Electron IPC
                        ▼
┌───────────────────────────────────────────────────────────────┐
│              MCP Servers (Main Process)                        │
│                                                               │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐          │
│  │ odei-neo4j  │  │odei-history │  │ odei-apple  │          │
│  └─────────────┘  └─────────────┘  └─────────────┘          │
└───────────────────────────────────────────────────────────────┘
```

### Implementation Details

**MCPClient Class** (`src/modules/mcpClient.js`):

```javascript
export class MCPClient {
  constructor(options = {}) {
    this.toolTimeoutMs = options.toolTimeoutMs || 10_000; // 10 seconds
    this.maxRetries = options.maxRetries ?? 1; // 1 retry by default
    this.backoffMs = options.backoffMs || 400; // Exponential backoff base

    // Concurrency control
    this.maxConcurrent = options.maxConcurrent ?? 6; // Max 6 concurrent requests
    this._activeRequests = 0;
    this._requestQueue = []; // FIFO queue

    // Queue timeout - prevent indefinite waits
    this.queueTimeoutMs = options.queueTimeoutMs || 30_000; // 30 seconds

    this.metricsListeners = new Set();
  }

  async callTool(server, toolName, params = {}) {
    // Track for freeze detection
    const endTracking = window.freezeDetector?.trackOperation?.(`MCP:${toolName}`, { server }) || (() => {});

    // SEMAPHORE: Wait for available slot
    await this._acquireSlot();

    const start = performance.now();
    let attempt = 0;
    let lastError;

    try {
      while (attempt <= this.maxRetries) {
        try {
          // Execute with timeout
          const result = await this.withTimeout(
            () => this.invoke(server, toolName, params),
            this.toolTimeoutMs,
            `Timeout: ${toolName}`
          );

          const latency = Math.round(performance.now() - start);
          this.emitMetrics({ server, toolName, latency, ok: true, attempt });
          return { result, latency };
        } catch (error) {
          lastError = error;
          const latency = Math.round(performance.now() - start);
          this.emitMetrics({ server, toolName, latency, ok: false, error, attempt });

          if (attempt === this.maxRetries) break;

          // Exponential backoff: 400ms, 800ms, 1600ms, ...
          await this.delay(this.backoffMs * (attempt + 1));
          attempt++;
        }
      }

      throw lastError;
    } finally {
      // Always release slot, even on error
      this._releaseSlot();
      endTracking();
    }
  }

  // Acquire semaphore slot - wait if at capacity
  async _acquireSlot() {
    if (this._activeRequests < this.maxConcurrent) {
      this._activeRequests++;
      return;
    }

    // Wait for slot with timeout
    return new Promise((resolve, reject) => {
      let resolved = false;

      const timeoutId = setTimeout(() => {
        if (!resolved) {
          resolved = true;
          // Remove from queue
          const idx = this._requestQueue.findIndex((item) => item.resolve === resolveWrapper);
          if (idx !== -1) this._requestQueue.splice(idx, 1);
          reject(new Error(`Semaphore queue timeout after ${this.queueTimeoutMs}ms`));
        }
      }, this.queueTimeoutMs);

      const resolveWrapper = () => {
        if (!resolved) {
          resolved = true;
          clearTimeout(timeoutId);
          resolve();
        }
      };

      this._requestQueue.push({ resolve: resolveWrapper, timeoutId });
    });
  }

  // Release semaphore slot - process next in queue
  _releaseSlot() {
    if (this._requestQueue.length > 0) {
      const next = this._requestQueue.shift();
      next.resolve(); // Transfer slot to waiting request
    } else {
      this._activeRequests--;
    }

    // Safety: prevent negative count
    if (this._activeRequests < 0) {
      console.warn('[mcpClient] activeRequests went negative, resetting to 0');
      this._activeRequests = 0;
    }
  }

  async withTimeout(fn, timeoutMs, message = 'Timeout') {
    let timer;
    try {
      return await Promise.race([
        fn(),
        new Promise((_, reject) => {
          timer = setTimeout(() => reject(new Error(message)), timeoutMs);
        }),
      ]);
    } finally {
      if (timer) clearTimeout(timer);
    }
  }

  delay(ms) {
    return new Promise((resolve) => setTimeout(resolve, ms));
  }
}

// Export singleton
export const mcpClient = new MCPClient();
```

### Design Principles

1. **Bounded Concurrency**: Never overwhelm MCP servers - max 6 concurrent requests
2. **Fail Fast**: 10-second timeout prevents indefinite hangs
3. **Resilience**: 1 automatic retry handles transient network errors
4. **Fairness**: FIFO queue ensures requests processed in order
5. **Observability**: Metrics track latency, success rate, queue depth
6. **Graceful Degradation**: On error, return partial results or cached data when possible

### MCP Server Separation

**Separate servers by domain** to avoid coupling:

- `odei-neo4j` - Graph database (Memory Atlas, entities, relationships)
- `odei-history` - Conversation persistence (agent transcripts, threading)
- `odei-apple` - Health data (HealthKit, workouts, sleep)
- `odei-gemini` - AI integration (Gemini API calls, embeddings)

**Why separate servers?**

- **Isolation** - Neo4j crash doesn't affect conversation history
- **Scaling** - Can add more health servers without touching graph
- **Permissions** - Apple server needs macOS entitlements, others don't
- **Simplicity** - Each server has single responsibility

## Consequences

### Positive Consequences

- **No more freezes** - Concurrency limit prevents overwhelming servers
- **Predictable timeouts** - 10-second max wait, no indefinite hangs
- **Resilient** - Automatic retry handles transient failures
- **Observable** - Metrics show performance issues immediately
- **Fair queuing** - FIFO ensures all requests eventually processed
- **Debuggable** - FreezeDetector integration shows which MCP calls are slow

### Negative Consequences

- **Queue wait time** - Requests may wait 30+ seconds if queue is full
- **Complexity** - Semaphore logic is harder to understand than simple Promise.all
- **Memory overhead** - Queue stores pending requests, can grow large under load
- **Starvation possible** - If 6 long-running requests block all slots, quick ones wait
- **Not truly concurrent** - Still runs on main thread, just limits active promises

### Neutral Consequences

- **Global semaphore** - Single mcpClient instance controls all MCP traffic
- **Fixed limits** - maxConcurrent=6 hardcoded, not dynamic based on load
- **Metrics optional** - Can listen to metrics, but not required

## Alternatives Considered

### Alternative 1: No Concurrency Control

**Description**: Let application code make unlimited concurrent MCP calls.

```javascript
// Unlimited concurrency - bad!
const results = await Promise.all([
  mcpClient.callTool('odei-neo4j', 'tool1', {}),
  mcpClient.callTool('odei-neo4j', 'tool2', {}),
  // ... 50 more calls
]);
```

**Pros**:

- Simple - no queue management
- Fast when server can handle load

**Cons**:

- **Server overload** - 50 concurrent requests crash Neo4j
- **Main thread freeze** - Waiting for 50 promises blocks event loop
- **Memory exhaustion** - Each request allocates buffers
- **No fairness** - First caller gets all resources

**Why rejected**: This was the original implementation. It caused 5-second freezes and server crashes.

### Alternative 2: Per-Server Semaphores

**Description**: Separate semaphore for each MCP server.

```javascript
const neo4jClient = new MCPClient({ maxConcurrent: 6 });
const historyClient = new MCPClient({ maxConcurrent: 3 });
const appleClient = new MCPClient({ maxConcurrent: 2 });
```

**Pros**:

- Tuned limits per server capacity
- Isolation between servers
- More total throughput (6+3+2=11 concurrent)

**Cons**:

- **Complexity** - Need to track multiple clients
- **Configuration burden** - Must tune each server limit
- **Uneven distribution** - Neo4j gets 6 slots, history gets 3 (why?)
- **Total still bounded** - Can't exceed sum of limits

**Why rejected**: Overengineering. Single global limit is simpler and sufficient. We can revisit if profiling shows need.

### Alternative 3: Worker Pool

**Description**: Run MCP calls in Web Workers to avoid main thread blocking.

```javascript
// Worker pool of 4 workers
const workerPool = new WorkerPool(4);
const result = await workerPool.exec('callMCPTool', { server, tool, params });
```

**Pros**:

- Truly concurrent - doesn't block main thread
- Better CPU utilization
- Can run 100+ tasks in parallel

**Cons**:

- **Electron IPC not worker-safe** - MCP bridge uses Electron IPC, which requires main thread
- **Serialization overhead** - Must serialize params and results (no functions, no DOM)
- **Complexity** - Need worker management, message passing, error handling
- **Not needed** - MCP calls are I/O-bound (network), not CPU-bound

**Why rejected**: Electron IPC bridge isn't accessible from workers. Would need major refactor. Semaphore is sufficient.

## Implementation Notes

### Key Files

- `/src/modules/mcpClient.js` - MCPClient implementation
- `/src/modules/UIManager.js` - Sidebar updates use mcpClient for MCP calls
- `/src/modules/MemoryViewManager.js` - Graph data fetching via mcpClient
- `/src/conversation-manager.js` - Conversation persistence via mcpClient

### Usage Pattern

```javascript
import { mcpClient } from './modules/mcpClient.js';

// Basic call with auto-retry, timeout, semaphore
const { result, latency } = await mcpClient.callTool('odei-neo4j', 'odei.neo4j.plan.backlog.v1', { limit: 10 });
console.log(`Fetched in ${latency}ms`);

// Listen to metrics
const unsubscribe = mcpClient.onMetrics((metric) => {
  console.log(`[${metric.server}] ${metric.toolName}: ${metric.latency}ms`);
  if (!metric.ok) {
    console.error('MCP call failed:', metric.error);
  }
});
```

### Performance Budgets

- **MCP call latency**: <200ms (fast), <1000ms (acceptable), >3000ms (investigate)
- **Queue depth**: <10 requests (healthy), 10-30 (degraded), >30 (critical)
- **Timeout**: 10 seconds max per call
- **Retry delay**: 400ms base, exponential backoff

### Error Handling

MCP calls can fail for various reasons:

- **Network error**: Server unreachable
- **Timeout**: Took longer than 10 seconds
- **Server error**: Tool threw exception
- **Queue timeout**: Waited 30+ seconds for slot

Always wrap MCP calls in try-catch:

```javascript
try {
  const { result } = await mcpClient.callTool('server', 'tool', params);
  // Use result
} catch (err) {
  console.error('[MyModule] MCP call failed:', err.message);
  // Degrade gracefully - show cached data, empty state, etc.
}
```

## References

- `/src/modules/mcpClient.js` - Implementation
- `/src/modules/FreezeDetector.js` - Integration with operation tracking
- [Semaphore Pattern](<https://en.wikipedia.org/wiki/Semaphore_(programming)>)
- [MCP Protocol Specification](https://modelcontextprotocol.io/docs)
- ADR-002: Freeze Detection - Explains FreezeDetector integration
- ADR-004: State Management - Explains Store singleton pattern
