# MCPClient API Reference

**Module**: `src/modules/mcpClient.js`
**Type**: Singleton
**Purpose**: MCP server communication with concurrency control, timeouts, and retries

## Overview

The `MCPClient` class provides a semaphore-based client for calling MCP (Model Context Protocol) tools with built-in concurrency limiting, automatic retries, timeout handling, and metrics collection.

## Import

```javascript
import { mcpClient } from './modules/mcpClient.js';
```

The `mcpClient` singleton is auto-instantiated with default options.

## Constructor Options

```typescript
interface MCPClientOptions {
  toolTimeoutMs?: number; // Timeout per tool call (default: 10000ms)
  maxRetries?: number; // Number of retries on failure (default: 1)
  backoffMs?: number; // Exponential backoff base (default: 400ms)
  maxConcurrent?: number; // Max concurrent requests (default: 6)
  queueTimeoutMs?: number; // Max wait time in queue (default: 30000ms)
}
```

**Defaults:**

```javascript
new MCPClient({
  toolTimeoutMs: 10000, // 10 seconds per call
  maxRetries: 1, // 1 automatic retry
  backoffMs: 400, // 400ms base backoff
  maxConcurrent: 6, // Max 6 concurrent requests
  queueTimeoutMs: 30000, // 30 second queue timeout
});
```

## Methods

### callTool()

Call an MCP tool with automatic retry, timeout, and semaphore control.

**Signature:**

```javascript
async callTool(
  server: string,
  toolName: string,
  params?: object
): Promise<{ result: any, latency: number }>
```

**Parameters:**

- `server` - MCP server name ('odei-neo4j', 'odei-history', 'odei-apple', etc.)
- `toolName` - Tool identifier (e.g., 'odei.neo4j.plan.backlog.v1')
- `params` - Tool parameters (optional, default: `{}`)

**Returns:**

- `Promise<{ result, latency }>` - Tool result and latency in milliseconds

**Throws:**

- `Error` - On timeout, network error, or after all retries exhausted

**Example:**

```javascript
try {
  const { result, latency } = await mcpClient.callTool('odei-neo4j', 'odei.neo4j.plan.backlog.v1', { limit: 10 });
  console.log(`Fetched in ${latency}ms:`, result);
} catch (error) {
  console.error('MCP call failed:', error.message);
}
```

**Behavior:**

1. **Acquire semaphore slot** - Waits if at `maxConcurrent` limit (FIFO queue)
2. **Track operation** - Registers with FreezeDetector for diagnostics
3. **Execute with timeout** - Calls tool via IPC bridge with 10s timeout
4. **Retry on failure** - Retries once with 400ms backoff
5. **Emit metrics** - Publishes latency, success/failure to listeners
6. **Release slot** - Always releases semaphore, even on error

**Notes:**

- Blocks if 6 concurrent requests are active (waits for slot)
- Timeout resets per retry attempt
- Metrics emitted for both success and failure

---

### onMetrics()

Subscribe to metrics events for monitoring and debugging.

**Signature:**

```javascript
onMetrics(listener: (metric: Metric) => void): () => void
```

**Parameters:**

- `listener` - Function called when metric is emitted

**Returns:**

- `Function` - Unsubscribe function

**Metric Structure:**

```typescript
interface Metric {
  server: string; // MCP server name
  toolName: string; // Tool identifier
  latency: number; // Call duration in ms
  ok: boolean; // Success flag
  attempt: number; // Retry attempt (0 = first try, 1 = retry)
  error?: Error; // Error object if failed
}
```

**Example:**

```javascript
const unsubscribe = mcpClient.onMetrics((metric) => {
  console.log(`[${metric.server}] ${metric.toolName}: ${metric.latency}ms`);

  if (!metric.ok) {
    console.error('Call failed:', metric.error);
  }

  if (metric.latency > 1000) {
    console.warn('Slow call detected!');
  }
});

// Cleanup
unsubscribe();
```

---

### abortAll()

Abort all pending requests - used during page unload.

**Signature:**

```javascript
abortAll(): void
```

**Behavior:**

- Clears request queue
- Rejects all queued promises (gracefully, not as errors)
- Resets active request counter
- Clears metrics listeners

**Example:**

```javascript
// In page unload handler
window.addEventListener('beforeunload', () => {
  mcpClient.abortAll();
});
```

**Notes:**

- Prevents unhandled promise rejections during unload
- Does not abort in-flight requests (those complete normally)
- Only affects queued requests waiting for semaphore slot

---

### withTimeout()

Execute a function with timeout. **Internal helper method.**

**Signature:**

```javascript
async withTimeout(
  fn: () => Promise<any>,
  timeoutMs: number,
  message?: string
): Promise<any>
```

**Parameters:**

- `fn` - Async function to execute
- `timeoutMs` - Timeout in milliseconds
- `message` - Error message on timeout (default: 'Timeout')

**Returns:**

- `Promise<any>` - Result of `fn()`

**Throws:**

- `Error` - If timeout reached before `fn()` resolves

**Example:**

```javascript
// Internal usage
const result = await mcpClient.withTimeout(() => fetch('/api/data'), 5000, 'API timeout');
```

---

### delay()

Delay execution by specified milliseconds. **Internal helper method.**

**Signature:**

```javascript
async delay(ms: number): Promise<void>
```

**Parameters:**

- `ms` - Milliseconds to delay

**Example:**

```javascript
// Internal usage for exponential backoff
await mcpClient.delay(400); // Wait 400ms
```

## Internal Properties

### Concurrency Control

```javascript
this.maxConcurrent = 6; // Max concurrent requests
this._activeRequests = 0; // Current active count
this._requestQueue = []; // FIFO queue of waiting requests
```

### Semaphore Implementation

**\_acquireSlot()** - Waits for available semaphore slot

```javascript
async _acquireSlot(): Promise<void>
```

**\_releaseSlot()** - Releases slot and processes next in queue

```javascript
_releaseSlot(): void
```

**Queue Structure:**

```javascript
{
  resolve: Function,   // Resolve promise when slot available
  timeoutId: number    // setTimeout ID for queue timeout
}
```

### Metrics

```javascript
this.metricsListeners = new Set(); // Metric subscribers
```

## Usage Patterns

### Basic Call

```javascript
import { mcpClient } from './modules/mcpClient.js';

const { result } = await mcpClient.callTool('odei-neo4j', 'odei.neo4j.search.v1', { query: 'test' });
```

### With Error Handling

```javascript
try {
  const { result, latency } = await mcpClient.callTool('odei-neo4j', 'odei.neo4j.plan.backlog.v1', { limit: 10 });

  console.log(`Fetched ${result.length} items in ${latency}ms`);
} catch (error) {
  if (error.message.includes('Timeout')) {
    console.error('MCP call timed out after 10s');
  } else if (error.message.includes('queue timeout')) {
    console.error('Request queue full, waited 30s+');
  } else {
    console.error('MCP call failed:', error);
  }

  // Degrade gracefully
  return { result: [], latency: 0 };
}
```

### Parallel Calls (Semaphore Limited)

```javascript
// These will execute max 6 at a time, rest queue up
const promises = tools.map((tool) => mcpClient.callTool('odei-neo4j', tool, {}));

const results = await Promise.all(promises);
```

### Performance Monitoring

```javascript
const metrics = [];

const unsub = mcpClient.onMetrics((metric) => {
  metrics.push(metric);

  // Alert on slow calls
  if (metric.latency > 3000) {
    console.warn(`Slow MCP call: ${metric.toolName} took ${metric.latency}ms`);
  }

  // Track failure rate
  const failures = metrics.filter((m) => !m.ok).length;
  const failureRate = failures / metrics.length;
  if (failureRate > 0.1) {
    console.error(`High failure rate: ${(failureRate * 100).toFixed(1)}%`);
  }
});
```

## Performance Budgets

### Latency Targets

- **Fast**: <200ms (most queries, cached data)
- **Acceptable**: 200-1000ms (complex queries, graph traversals)
- **Slow**: 1000-3000ms (large result sets, aggregations)
- **Critical**: >3000ms (investigate and optimize)

### Queue Depth

- **Healthy**: 0-5 queued requests
- **Degraded**: 5-15 queued requests
- **Critical**: >15 queued requests (consider scaling)

### Concurrency

- **Current limit**: 6 concurrent requests (balanced for Electron/Neo4j)
- **Too low**: <3 (underutilizes server capacity)
- **Too high**: >10 (overwhelms Neo4j, causes freezes)

## Error Handling

### Common Errors

**Timeout Error:**

```javascript
Error: Timeout: odei.neo4j.search.v1;
```

- **Cause**: Call took >10 seconds
- **Solution**: Optimize query, add indexes, increase timeout

**Queue Timeout Error:**

```javascript
Error: Semaphore queue timeout after 30000ms
```

- **Cause**: Waited 30+ seconds for semaphore slot
- **Solution**: Too many concurrent calls, reduce load

**Network Error:**

```javascript
Error: Failed to fetch
```

- **Cause**: IPC bridge failed, server unreachable
- **Solution**: Restart MCP server, check Electron IPC

**Server Error:**

```javascript
Error: Neo4j error: ...
```

- **Cause**: MCP tool threw exception
- **Solution**: Fix query, check Neo4j logs

### Error Recovery Strategies

```javascript
async function robustMCPCall(server, tool, params) {
  let retries = 3;
  let delay = 1000;

  while (retries > 0) {
    try {
      return await mcpClient.callTool(server, tool, params);
    } catch (error) {
      retries--;

      if (retries === 0) {
        // Final retry failed - return cached data
        console.error('All retries exhausted, using cache');
        return { result: getCachedData(tool), latency: 0 };
      }

      // Exponential backoff
      await new Promise((r) => setTimeout(r, delay));
      delay *= 2;
    }
  }
}
```

## Debugging

### Check Queue State

```javascript
// In DevTools console
window.mcpClient._activeRequests; // Current active count
window.mcpClient._requestQueue.length; // Queued requests
window.mcpClient.maxConcurrent; // Max concurrent limit
```

### Simulate High Load

```javascript
// Flood with requests to test semaphore
const promises = Array(20)
  .fill(null)
  .map((_, i) => mcpClient.callTool('odei-neo4j', 'test.tool', { index: i }));

// Should process max 6 at a time
await Promise.all(promises);
```

### Track All Calls

```javascript
const calls = [];

mcpClient.onMetrics((metric) => {
  calls.push({
    tool: metric.toolName,
    latency: metric.latency,
    ok: metric.ok,
    timestamp: Date.now(),
  });

  console.table(calls.slice(-10)); // Last 10 calls
});
```

## Best Practices

### DO

- Wrap calls in try-catch
- Monitor metrics for performance issues
- Use meaningful tool names
- Keep params serializable (no functions, DOM nodes)
- Degrade gracefully on errors

### DON'T

- Call from tight loops without await
- Ignore errors (always handle)
- Make >100 parallel calls (queue will overflow)
- Mutate params object during call
- Forget to unsubscribe from metrics

## Testing

### Mock MCP Client

```javascript
jest.mock('./modules/mcpClient.js', () => ({
  mcpClient: {
    callTool: jest.fn().mockResolvedValue({
      result: { data: 'test' },
      latency: 100,
    }),
  },
}));

test('handles MCP data', async () => {
  const { result } = await mcpClient.callTool('server', 'tool', {});
  expect(result.data).toBe('test');
});
```

### Test Timeout

```javascript
test('throws on timeout', async () => {
  mcpClient.toolTimeoutMs = 100; // Short timeout

  await expect(mcpClient.callTool('slow-server', 'slow-tool', {})).rejects.toThrow('Timeout');
});
```

## Related

- [ADR-003: MCP Server Architecture](../adr/003-mcp-server-architecture.md) - Design rationale
- [ADR-002: Freeze Detection](../adr/002-freeze-detection.md) - FreezeDetector integration
- [Store API](./Store.md) - State management
