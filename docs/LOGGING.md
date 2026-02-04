# ODEI Logging and Observability

Comprehensive guide to ODEI's logging infrastructure for debugging production issues and monitoring system health.

## Overview

ODEI implements a structured logging system with:

- **Structured JSON logging** — Machine-readable logs with consistent format
- **Log rotation** — Automatic rotation to prevent disk space issues
- **Correlation IDs** — Trace requests across distributed components
- **Performance metrics** — Track tool invocations, response times, errors
- **Debug endpoints** — HTTP endpoints for health checks and metrics
- **Multi-process logging** — Main, renderer, and MCP server logs

---

## Architecture

### Components

```
┌─────────────────────────────────────────────────────────┐
│                    ODEI Application                      │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  Main Process         Renderer Process      MCP Servers │
│  ┌──────────┐        ┌──────────┐         ┌──────────┐ │
│  │ Logger   │◄──IPC──│ Console  │         │ pino     │ │
│  │  +       │        │ Capture  │         │ Logger   │ │
│  │ File     │        └──────────┘         │  +       │ │
│  │ Rotation │                             │ Metrics  │ │
│  └────┬─────┘                             └────┬─────┘ │
│       │                                        │        │
│       ▼                                        ▼        │
│  ┌────────────────────────────────────────────────┐    │
│  │         Log Files (~/Library/Application       │    │
│  │         Support/ODEI/logs/)                    │    │
│  │                                                 │    │
│  │  - main.log           (Main process)           │    │
│  │  - main-error.log     (Errors only)            │    │
│  │  - renderer.log       (Renderer process)       │    │
│  │                                                 │    │
│  │  MCP server logs (stderr):                     │    │
│  │  - odei-neo4j stderr                           │    │
│  │  - odei-history stderr                         │    │
│  └────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
```

### Log Levels

| Level   | Priority | Usage                                  |
| ------- | -------- | -------------------------------------- |
| `trace` | 10       | Very detailed debugging (rarely used)  |
| `debug` | 20       | Detailed debugging information         |
| `info`  | 30       | General informational messages         |
| `warn`  | 40       | Warning messages                       |
| `error` | 50       | Error messages                         |
| `fatal` | 60       | Fatal errors (application will crash)  |

---

## Log File Locations

### Electron App

Log files are stored in the application's user data directory:

**macOS:**
```
~/Library/Application Support/ODEI/logs/
```

**Linux:**
```
~/.config/ODEI/logs/
```

**Windows:**
```
%APPDATA%\ODEI\logs\
```

### Log Files

| File              | Contents                              | Rotation |
| ----------------- | ------------------------------------- | -------- |
| `main.log`        | Main process (all levels)             | 10MB     |
| `main-error.log`  | Main process errors only              | 10MB     |
| `renderer.log`    | Renderer process logs                 | 10MB     |

Rotated files are numbered: `main.log.1`, `main.log.2`, etc. (max 5 files)

### MCP Server Logs

MCP servers log to **stderr** (not files) to avoid corrupting JSON-RPC protocol on stdout.

To capture MCP logs:

```bash
# Development mode (pretty print)
NODE_ENV=development node servers/odei-neo4j/dist/index.js 2> neo4j.log

# Production mode (JSON)
node servers/odei-neo4j/dist/index.js 2> neo4j.log
```

---

## Configuration

### Environment Variables

| Variable                  | Default | Description                                    |
| ------------------------- | ------- | ---------------------------------------------- |
| `LOG_LEVEL`               | `info`  | Minimum log level to output                    |
| `NODE_ENV`                | —       | Set to `development` for pretty printing       |
| `ODEI_DEBUG_ENDPOINTS`    | `0`     | Enable debug HTTP endpoints (set to `1`)       |
| `ODEI_DEBUG_PORT`         | random  | Port for debug endpoints                       |

### Example Configuration

**Development (.env):**
```bash
LOG_LEVEL=debug
NODE_ENV=development
ODEI_DEBUG_ENDPOINTS=1
ODEI_DEBUG_PORT=9001
```

**Production (.env):**
```bash
LOG_LEVEL=info
NODE_ENV=production
ODEI_DEBUG_ENDPOINTS=0
```

---

## Usage

### Electron Main Process

```javascript
const { logger, getModuleLogger } = require('./logger');

// Use main logger
logger.info('Application starting');
logger.error('Database connection failed', { uri: 'neo4j://localhost' }, error);

// Create module-specific logger
const agentLogger = getModuleLogger('agent-manager');
agentLogger.debug('Agent started', { agent: 'discuss' });

// Time an operation
await logger.time('loadConfiguration', async () => {
  return loadConfig();
});
```

### Electron Renderer Process

```javascript
// Use window.odei.logger (exposed via preload)
window.odei.logger.info('View loaded', { view: 'dashboard' });
window.odei.logger.error('Failed to fetch data', { error: err.message });

// Read recent logs
const logs = await window.odei.logger.readRecentLogs('main.log', 100);
console.log('Recent logs:', logs);
```

### MCP Servers (TypeScript)

```typescript
import { createMCPLogger, MetricsCollector, ToolTimer } from '../shared-logger.js';

// Create logger
const logger = createMCPLogger({
  name: 'odei-neo4j',
  level: process.env.LOG_LEVEL,
  isDevelopment: process.env.NODE_ENV === 'development',
});

// Initialize metrics
const metrics = new MetricsCollector();
const toolTimer = new ToolTimer(logger, metrics);

// Time a tool invocation
const correlationId = toolTimer.start('odei.neo4j.node.get.v1');
try {
  const result = await getNode(nodeId);
  toolTimer.end(correlationId, true);
  return result;
} catch (error) {
  toolTimer.end(correlationId, false, error);
  throw error;
}

// Get metrics summary
const summary = metrics.getSummary();
logger.info({ metrics: summary }, 'Metrics summary');
```

---

## Debug Endpoints

MCP servers can expose HTTP debug endpoints for monitoring.

### Enable Debug Endpoints

Set environment variables:
```bash
ODEI_DEBUG_ENDPOINTS=1
ODEI_DEBUG_PORT=9001  # Optional, defaults to random port
```

### Available Endpoints

#### `GET /health`

Health check endpoint.

**Response:**
```json
{
  "status": "healthy",
  "server": "odei-neo4j",
  "version": "0.1.0",
  "timestamp": "2025-12-25T10:00:00.000Z",
  "uptime": 3600,
  "memory": {
    "heapUsed": 45,
    "heapTotal": 64,
    "rss": 128
  }
}
```

#### `GET /metrics`

Tool invocation metrics.

**Response:**
```json
{
  "uptime": 3600000,
  "totalInvocations": 1234,
  "totalErrors": 12,
  "errorRate": 0.97,
  "tools": {
    "odei.neo4j.node.get.v1": {
      "invocations": 456,
      "errors": 5,
      "errorRate": 1.09,
      "avgDuration": 45.2,
      "minDuration": 12,
      "maxDuration": 234
    }
  }
}
```

#### `GET /log-level`

Get current log level.

**Response:**
```json
{
  "level": "info"
}
```

#### `POST /log-level`

Change log level at runtime.

**Request:**
```json
{
  "level": "debug"
}
```

**Response:**
```json
{
  "oldLevel": "info",
  "newLevel": "debug"
}
```

#### `GET /status`

Comprehensive status information.

**Response:**
```json
{
  "server": "odei-neo4j",
  "version": "0.1.0",
  "status": "running",
  "timestamp": "2025-12-25T10:00:00.000Z",
  "uptime": {
    "seconds": 3600,
    "formatted": "1h"
  },
  "memory": {
    "heapUsed": 45,
    "heapTotal": 64,
    "rss": 128,
    "external": 2
  },
  "process": {
    "pid": 12345,
    "platform": "darwin",
    "nodeVersion": "v20.11.0"
  },
  "logging": {
    "level": "info"
  },
  "metrics": { ... }
}
```

---

## Debugging Production Issues

### Step 1: Check Log Files

```bash
# Navigate to log directory (macOS)
cd ~/Library/Application\ Support/ODEI/logs/

# View recent errors
tail -100 main-error.log

# View all logs
tail -500 main.log

# Search for specific errors
grep "ERROR" main.log | tail -50
```

### Step 2: Check MCP Server Status

If MCP servers are running with debug endpoints enabled:

```bash
# Health check
curl http://localhost:9001/health

# Metrics
curl http://localhost:9001/metrics

# Full status
curl http://localhost:9001/status
```

### Step 3: Enable Verbose Logging

Temporarily increase log level:

```bash
# For Electron app
export LOG_LEVEL=debug

# For MCP servers (via debug endpoint)
curl -X POST http://localhost:9001/log-level \
  -H "Content-Type: application/json" \
  -d '{"level": "debug"}'
```

### Step 4: Analyze Correlation IDs

Find related log entries using correlation IDs:

```bash
# Extract correlation ID from error
grep "correlationId" main-error.log | tail -1

# Find all logs for that request
grep "1735128000000-abc123xyz" main.log
```

### Step 5: Check Memory Usage

Monitor memory trends:

```bash
# Filter memory logs
grep "Memory usage" main.log | tail -20

# Parse JSON logs
cat main.log | jq 'select(.msg == "Memory usage (MB)")'
```

---

## Common Issues

### Issue: Logs Not Appearing

**Symptoms:**
- No log files created
- Missing log entries

**Solutions:**

1. Check log directory exists:
   ```bash
   ls -la ~/Library/Application\ Support/ODEI/logs/
   ```

2. Check permissions:
   ```bash
   chmod -R 755 ~/Library/Application\ Support/ODEI/logs/
   ```

3. Check log level is not too restrictive:
   ```bash
   export LOG_LEVEL=debug
   ```

### Issue: MCP Logs Missing

**Symptoms:**
- MCP tools failing silently
- No stderr output

**Solutions:**

1. MCP servers log to **stderr**, not stdout or files
2. Redirect stderr to capture logs:
   ```bash
   node servers/odei-neo4j/dist/index.js 2> mcp-neo4j.log
   ```

3. Enable debug mode:
   ```bash
   NODE_ENV=development node servers/odei-neo4j/dist/index.js
   ```

### Issue: High Log Volume

**Symptoms:**
- Log files growing too fast
- Disk space warnings

**Solutions:**

1. Increase log level to reduce volume:
   ```bash
   export LOG_LEVEL=warn
   ```

2. Decrease rotation size (edit `/src/utils/logger.js`):
   ```javascript
   maxFileSize: 5 * 1024 * 1024, // 5MB instead of 10MB
   maxFiles: 3, // Keep 3 files instead of 5
   ```

3. Enable log cleanup on startup (automatic)

### Issue: Performance Impact

**Symptoms:**
- Slow application performance
- High CPU usage

**Solutions:**

1. Reduce log level:
   ```bash
   export LOG_LEVEL=error
   ```

2. Disable debug endpoints:
   ```bash
   export ODEI_DEBUG_ENDPOINTS=0
   ```

3. Disable metrics collection (edit server code):
   ```typescript
   const logger = createMCPLogger({
     enableMetrics: false, // Disable metrics
   });
   ```

---

## Best Practices

### 1. Use Appropriate Log Levels

- `trace` — Rarely used, very verbose
- `debug` — Development debugging
- `info` — Normal operation (default)
- `warn` — Unexpected but recoverable
- `error` — Errors requiring attention
- `fatal` — Crash-level errors

### 2. Include Context

Always include relevant context data:

```javascript
// Bad
logger.error('Failed to save');

// Good
logger.error('Failed to save node', { nodeId, type, error: err.message });
```

### 3. Use Correlation IDs

Track distributed operations:

```javascript
const correlationId = Logger.generateCorrelationId();
logger.info('Processing request', { correlationId, operation });
// ... later ...
logger.info('Request completed', { correlationId, duration });
```

### 4. Sanitize Sensitive Data

Never log passwords, tokens, or secrets:

```javascript
// Bad
logger.info('Config loaded', { neo4jPassword: password });

// Good
logger.info('Config loaded', { neo4jUri, neo4jPassword: '***REDACTED***' });
```

### 5. Monitor Metrics

Regularly review metrics to identify issues:

```bash
# Check error rates
curl http://localhost:9001/metrics | jq '.errorRate'

# Check slow operations
curl http://localhost:9001/metrics | jq '.tools[] | select(.avgDuration > 1000)'
```

---

## Log Format

### JSON Format (Production)

```json
{
  "level": 30,
  "time": "2025-12-25T10:00:00.000Z",
  "pid": 12345,
  "name": "odei-neo4j",
  "msg": "Node retrieved",
  "correlationId": "1735128000000-abc123xyz",
  "nodeId": "foundation-value-001",
  "duration": 45
}
```

### Pretty Format (Development)

```
[10:00:00] INFO  odei-neo4j: Node retrieved [1735128000000-abc123xyz] (45ms)
  { nodeId: 'foundation-value-001' }
```

---

## Metrics

### Collected Metrics

- **Tool invocations** — Count per tool
- **Error count** — Total and per tool
- **Error rate** — Percentage of failed invocations
- **Response times** — Min, max, average per tool
- **Memory usage** — Heap, RSS, external
- **Uptime** — Process uptime

### Accessing Metrics

**Via HTTP endpoint:**
```bash
curl http://localhost:9001/metrics
```

**Via code:**
```typescript
const summary = metrics.getSummary();
logger.info({ metrics: summary }, 'Metrics summary');
```

**Via main process:**
```javascript
const metrics = logger.getMetrics();
console.log('Metrics:', metrics);
```

---

## Advanced Topics

### Custom Loggers

Create custom loggers for specific modules:

```javascript
const myLogger = logger.child({ module: 'my-module', customField: 'value' });
myLogger.info('Module initialized');
// Output includes: { module: 'my-module', customField: 'value', ... }
```

### Timing Operations

Use the built-in timer:

```javascript
await logger.time('databaseQuery', async () => {
  return db.query('MATCH (n) RETURN n');
});
// Automatically logs start, completion, duration, and errors
```

### Console Capture

Renderer console output is automatically captured:

```javascript
// In renderer
console.log('User action:', action);
console.error('UI error:', error);

// Automatically forwarded to main.log with renderer context
```

---

## Troubleshooting

### Enable Maximum Verbosity

```bash
# Electron app
export LOG_LEVEL=trace
export NODE_ENV=development
export ODEI_DEBUG_ENDPOINTS=1

# Restart app
npm start
```

### Capture All Output

```bash
# Redirect all output to files
npm start > output.log 2> error.log
```

### Analyze Performance

```bash
# Get metrics
curl http://localhost:9001/metrics > metrics.json

# Find slow operations
cat metrics.json | jq '.tools[] | select(.avgDuration > 100)'
```

---

## Support

For issues with logging:

1. Check this documentation
2. Review log files in `~/Library/Application Support/ODEI/logs/`
3. Enable debug mode and debug endpoints
4. Check metrics for anomalies
5. Search existing logs for correlation IDs

**Log Directory:** `~/Library/Application Support/ODEI/logs/` (macOS)
**Debug Endpoints:** `http://localhost:9001/` (when enabled)
**Environment:** Set `LOG_LEVEL=debug` for verbose output
