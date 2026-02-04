# ODEI Conductor Output Management

## Overview

Robust output management system for ODEI Conductor that solves PTY output streaming issues with chunking, backpressure, file-based storage, and rate limiting.

## Problem Statement

PTY output streaming causes:

- **Huge data volumes** from verbose CLI tools
- **UI lag** when WebSocket receives too much data
- **Memory pressure** from unbounded buffers
- **WebSocket drops** when client can't keep up

## Solution Architecture

### 1. Output Chunking

**Strategy:** Aggregate output before sending over WebSocket

- **Chunk size:** 8KB (configurable 4-16KB range)
- **Batch timer:** 100ms intervals
- **Benefit:** Reduces WebSocket message overhead by 10-100x

```javascript
// Instead of sending every byte immediately:
ws.send({ type: 'output', data: 'H' });
ws.send({ type: 'output', data: 'e' });
ws.send({ type: 'output', data: 'l' });
// ...

// We batch into chunks:
ws.send({
  type: 'output_chunk',
  chunks: [
    { seq: 1, data: 'Hello\n' },
    { seq: 2, data: 'World\n' },
    // ... more chunks
  ],
});
```

### 2. Backpressure Handling

**Strategy:** When conductor can't keep up, buffer to file and notify client

- **Detection:** Monitor bytes sent per second
- **Action:** Write overflow to file, send `output_overflow` event
- **Recovery:** Client uses `getOutput` tool to fetch from file

```javascript
if (bytesThisSecond > maxRateKBps * 1024) {
  // Write to file (always happens)
  outputManager.write('stdout', data);

  // Skip WebSocket, send overflow notification
  ws.send({
    type: 'output_overflow',
    message: 'Output rate limit exceeded. Use getOutput to retrieve full log.',
  });
}
```

### 3. File-Based Storage

**Strategy:** All output persisted to disk, always

**Location:**

```
~/.odei/conductor/runs/{taskId}/
  ├── output.log    # NDJSON format
  └── meta.json     # Run metadata
```

**Format (output.log):**

```json
{"seq":1,"ts":1733673600000,"type":"stdout","data":"Starting task...\n"}
{"seq":2,"ts":1733673600100,"type":"stdout","data":"Processing...\n"}
{"seq":3,"ts":1733673600200,"type":"stderr","data":"Warning: deprecated API\n"}
```

**Format (meta.json):**

```json
{
  "runId": "task_1733673600000_a1b2c3d4",
  "createdAt": 1733673600000,
  "closedAt": 1733673700000,
  "totalBytes": 1048576,
  "totalChunks": 256,
  "version": 1
}
```

### 4. Tail Tool

**Strategy:** MCP tool to read output on-demand

**Tool:** `odei.conductor.getOutput.v1`

**Parameters:**

- `taskId` or `runId` - Which task to read from (file-based)
- `agentId` - Which agent to read from (real-time, legacy)
- `nLines` - Number of lines to retrieve (default: 100)
- `nBytes` - Max bytes to read (default: 1MB)

**Usage:**

```javascript
// Get last 100 lines of completed task
const output = await tools.getOutput({ taskId: 'task_123', nLines: 100 });

// Get full output (up to 1MB)
const fullOutput = await tools.getOutput({ taskId: 'task_123', nBytes: 1048576 });

// Real-time agent output (legacy)
const realtimeOutput = await tools.getOutput({ agentId: 'execute', nLines: 50 });
```

### 5. Rate Limiting

**Configuration:**

- **Max throughput:** 100KB/s per agent to WebSocket (configurable)
- **If exceeded:** Write to file only, send `output_overflow` event
- **UI response:** "Output truncated, run getOutput for full log"

**Per-agent tracking:**

```javascript
{
  bytesThisSecond: 85000,     // Current rate
  maxBytes: 102400,           // 100KB limit
  secondStart: 1733673600000  // Reset every second
}
```

## Implementation

### OutputManager Class

**Location:** `/Users/ai/ODEI/electron/output-manager.js`

**Methods:**

```javascript
// Create manager for a task
const manager = new OutputManager(taskId, {
  chunkSize: 8192,
  flushInterval: 100,
  maxRateKBps: 100,
  outputDir: '~/.odei/conductor/runs',
});

// Write output
manager.write('stdout', data);
manager.write('stderr', errorData);

// Get buffered chunk for WebSocket sending
const chunk = manager.flush();
if (chunk && !chunk.overflow) {
  ws.send({ type: 'output_chunk', ...chunk });
} else if (chunk && chunk.overflow) {
  ws.send({ type: 'output_overflow', ...chunk });
}

// Close when task completes
manager.close();

// Static methods for reading
const tail = OutputManager.getTail(taskId, 100);
const meta = OutputManager.getMeta(taskId);
const runs = OutputManager.listRuns(50);
```

**Events:**

```javascript
// Emitted when chunk ready for WebSocket
manager.on('chunk', (chunk) => {
  if (chunk.overflow) {
    // Rate limit exceeded
  } else {
    // Send chunks to WebSocket
  }
});

// Emitted when rate limit triggered
manager.on('ratelimit', ({ bytesThisSecond, maxBytes }) => {
  console.log(`Rate limit: ${bytesThisSecond}/${maxBytes}`);
});

// Emitted when manager closed
manager.on('closed', ({ runId, totalBytes }) => {
  console.log(`Closed: ${runId}, ${totalBytes} bytes`);
});
```

### Integration with Conductor

**File:** `/Users/ai/ODEI/electron/conductor-server.js`

**Changes:**

1. **Create OutputManager per task:**

```javascript
// In executeTask()
const outputManager = new OutputManager(taskId);
this.outputManagers.set(taskId, outputManager);

// Subscribe to chunks
outputManager.on('chunk', (chunk) => {
  if (chunk.overflow) {
    this.broadcast({ type: 'output_overflow', taskId, ...chunk });
  } else {
    this.broadcast({ type: 'output_chunk', taskId, ...chunk });
  }
});
```

2. **Write agent output to manager:**

```javascript
// In handleAgentOutput()
const activeTasks = this.taskStore.getActive().filter((t) => t.agentId === agentId);
if (activeTasks.length > 0) {
  const outputManager = this.outputManagers.get(activeTasks[0].id);
  if (outputManager) {
    outputManager.write('stdout', data);
  }
}
```

3. **Close manager on task completion:**

```javascript
// In completeTask()
const outputManager = this.outputManagers.get(taskId);
if (outputManager) {
  outputManager.close();
  this.outputManagers.delete(taskId);
}
```

4. **Add getTail handler:**

```javascript
// New WebSocket message handler
handleGetOutputTail(ws, msg) {
  const { taskId, nLines, nBytes } = msg;
  const chunks = OutputManager.getTail(taskId, nLines, nBytes);
  const meta = OutputManager.getMeta(taskId);

  this.sendToClient(ws, {
    type: 'output_tail',
    taskId,
    chunks,
    meta
  });
}
```

### MCP Tool Updates

**File:** `/Users/ai/ODEI/servers/odei-conductor/src/tools/getOutput.ts`

**Enhanced to support two modes:**

1. **File-based (recommended):**
   - Provide `taskId` or `runId`
   - Reads from persisted files
   - No truncation, complete logs
   - Works even if real-time streaming failed

2. **Real-time (legacy):**
   - Provide `agentId`
   - Reads from in-memory buffer
   - Limited to last N lines
   - May miss data if rate-limited

## Configuration

**Environment Variables:**

```bash
# Conductor output directory (default: ~/.odei/conductor/runs)
ODEI_OUTPUT_DIR=/custom/path

# Max rate limit per agent (KB/s, default: 100)
ODEI_MAX_RATE_KBPS=100

# Chunk size (bytes, default: 8192)
ODEI_CHUNK_SIZE=8192

# Flush interval (ms, default: 100)
ODEI_FLUSH_INTERVAL=100

# Max log age for cleanup (days, default: 30)
ODEI_MAX_LOG_AGE_DAYS=30
```

**Runtime Configuration:**

```javascript
const outputManager = new OutputManager(taskId, {
  chunkSize: 16384, // Larger chunks
  flushInterval: 50, // Faster flushing
  maxRateKBps: 200, // Higher rate limit
  outputDir: '/custom/path',
  maxLogAgeDays: 7, // Cleanup after 7 days
});
```

## Maintenance

### Log Cleanup

**Automatic cleanup** (recommended):

```javascript
// In conductor server startup
setInterval(
  () => {
    const { removed, errors } = OutputManager.cleanup(30); // 30 days
    console.log(`Cleaned up ${removed} old runs`);
  },
  24 * 60 * 60 * 1000
); // Daily
```

**Manual cleanup:**

```bash
# Remove logs older than 30 days
node -e "require('./electron/output-manager').OutputManager.cleanup(30)"
```

### Monitoring

**Disk usage:**

```bash
du -sh ~/.odei/conductor/runs
```

**Active runs:**

```javascript
const runs = OutputManager.listRuns(50);
console.log(`Active runs: ${runs.length}`);
console.log(`Total size: ${runs.reduce((sum, r) => sum + r.totalBytes, 0)} bytes`);
```

**Rate limit metrics:**

```javascript
outputManager.on('ratelimit', ({ bytesThisSecond }) => {
  metrics.increment('conductor.ratelimit');
  metrics.gauge('conductor.bytes_per_second', bytesThisSecond);
});
```

## Benefits

### Performance

- **10-100x reduction** in WebSocket messages
- **No memory pressure** from unbounded buffers
- **Graceful degradation** under load

### Reliability

- **Complete logs** always available
- **No data loss** even with network issues
- **Recovery** from WebSocket drops

### Scalability

- **Multiple concurrent tasks** without interference
- **Rate limiting** per agent prevents resource exhaustion
- **File-based storage** scales to large outputs

### Developer Experience

- **Simple API** for retrieving logs
- **Flexible querying** (by task, agent, time range)
- **Clear error messages** when rate-limited

## Testing

**Run tests:**

```bash
cd /Users/ai/ODEI/electron
node output-manager.test.js
```

**Test coverage:**

- ✓ Basic write and flush
- ✓ File storage verification
- ✓ Rate limiting
- ✓ Tail reading
- ✓ Metadata retrieval
- ✓ Run listing
- ✓ Cleanup

## Migration

**For orchestration clients:**

1. **Old way (real-time only):**

```javascript
const output = await conductor.getOutput({ agentId: 'execute' });
// Limited to in-memory buffer
// May miss output if rate-limited
```

2. **New way (file-based):**

```javascript
const output = await conductor.getOutput({ taskId: taskId });
// Complete logs from file
// Works even if real-time streaming dropped
```

**For UI:**

1. **Handle `output_overflow` events:**

```javascript
ws.on('message', (msg) => {
  if (msg.type === 'output_overflow') {
    console.log('Output rate-limited, use getOutput to retrieve full log');
    // Show UI notification with getOutput button
  }
});
```

2. **Use file-based retrieval for completed tasks:**

```javascript
// When task completes, always fetch from file for complete log
if (task.state === 'completed') {
  const output = await conductor.getOutput({ taskId: task.id });
}
```

## Future Enhancements

1. **Compression:** Gzip old log files
2. **Streaming:** Stream large log files in chunks
3. **Search:** Full-text search across logs
4. **Export:** Export logs to external systems
5. **Visualization:** Real-time log viewer in UI
6. **Filtering:** Filter by log level, timestamp, pattern
7. **Aggregation:** Combine logs from multiple agents

## References

- OutputManager: `/Users/ai/ODEI/electron/output-manager.js`
- Conductor Server: `/Users/ai/ODEI/electron/conductor-server.js`
- MCP Tool: `/Users/ai/ODEI/servers/odei-conductor/src/tools/getOutput.ts`
- Tests: `/Users/ai/ODEI/electron/output-manager.test.js`
