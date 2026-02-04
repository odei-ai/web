# Logging Quick Start Guide

Quick reference for using ODEI's logging system.

## TL;DR

```bash
# Enable verbose logging
export LOG_LEVEL=debug
export NODE_ENV=development
export ODEI_DEBUG_ENDPOINTS=1

# View logs (macOS)
tail -f ~/Library/Application\ Support/ODEI/logs/main.log

# Check server health
curl http://localhost:9001/health

# Get metrics
curl http://localhost:9001/metrics
```

## Common Use Cases

### 1. Debug Application Startup Issues

```bash
# Enable debug logging
export LOG_LEVEL=debug

# Start app
npm start

# Watch logs
tail -f ~/Library/Application\ Support/ODEI/logs/main.log
```

### 2. Debug MCP Server Issues

```bash
# Run MCP server with debug output
cd servers/odei-neo4j
NODE_ENV=development LOG_LEVEL=debug node dist/index.js 2>&1 | tee mcp.log
```

### 3. Monitor Performance

```bash
# Enable debug endpoints
export ODEI_DEBUG_ENDPOINTS=1

# Start app, then check metrics
curl http://localhost:9001/metrics | jq '.tools'

# Find slow operations
curl http://localhost:9001/metrics | jq '.tools[] | select(.avgDuration > 100)'
```

### 4. Capture Renderer Errors

```javascript
// Console errors are automatically captured
try {
  // Your code
} catch (error) {
  window.odei.logger.error('Operation failed', {
    operation: 'loadData',
    error: error.message
  });
}
```

### 5. Change Log Level at Runtime

```bash
# Check current level
curl http://localhost:9001/log-level

# Change to debug
curl -X POST http://localhost:9001/log-level \
  -H "Content-Type: application/json" \
  -d '{"level": "debug"}'
```

## Code Examples

### Main Process

```javascript
const { logger, getModuleLogger } = require('./logger');

// Basic logging
logger.info('Server started', { port: 3000 });
logger.error('Database error', { query }, error);

// Module-specific logger
const dbLogger = getModuleLogger('database');
dbLogger.debug('Query executed', { rows: result.length });

// Time an operation
await logger.time('startup', async () => {
  await initializeDatabase();
  await startServer();
});
```

### Renderer Process

```javascript
// Use window.odei.logger
window.odei.logger.info('Page loaded', { route: '/dashboard' });
window.odei.logger.warn('Slow operation', { duration: 1500 });
window.odei.logger.error('API error', { endpoint: '/api/data', error });

// Read logs
const recentLogs = await window.odei.logger.readRecentLogs('main.log', 50);
console.table(recentLogs);
```

### MCP Server

```typescript
import { createMCPLogger, ToolTimer, MetricsCollector } from './shared-logger.js';

// Create logger
const logger = createMCPLogger({
  name: 'my-server',
  level: process.env.LOG_LEVEL,
  isDevelopment: process.env.NODE_ENV === 'development',
});

// Initialize metrics
const metrics = new MetricsCollector();
const timer = new ToolTimer(logger, metrics);

// Time a tool
const correlationId = timer.start('my.tool');
try {
  const result = await executeTool();
  timer.end(correlationId, true);
  return result;
} catch (error) {
  timer.end(correlationId, false, error);
  throw error;
}

// Log metrics periodically
setInterval(() => {
  logger.info({ metrics: metrics.getSummary() }, 'Metrics');
}, 60000);
```

## Environment Variables

| Variable                | Example           | Description                  |
| ----------------------- | ----------------- | ---------------------------- |
| `LOG_LEVEL`             | `debug`           | Minimum log level            |
| `NODE_ENV`              | `development`     | Enable pretty printing       |
| `ODEI_DEBUG_ENDPOINTS`  | `1`               | Enable HTTP debug endpoints  |
| `ODEI_DEBUG_PORT`       | `9001`            | Debug endpoint port          |

## Log Levels

From most to least verbose:

1. `trace` — Very detailed (rarely used)
2. `debug` — Development debugging
3. `info` — Normal operation (default)
4. `warn` — Warnings
5. `error` — Errors
6. `fatal` — Fatal errors

## Debug Endpoints

When `ODEI_DEBUG_ENDPOINTS=1`:

```bash
# Health check
curl http://localhost:9001/health

# Metrics
curl http://localhost:9001/metrics

# Status
curl http://localhost:9001/status

# Get log level
curl http://localhost:9001/log-level

# Set log level
curl -X POST http://localhost:9001/log-level \
  -H "Content-Type: application/json" \
  -d '{"level": "debug"}'
```

## Finding Logs

### macOS
```bash
cd ~/Library/Application\ Support/ODEI/logs/
ls -lh
```

### Linux
```bash
cd ~/.config/ODEI/logs/
ls -lh
```

### Windows
```powershell
cd %APPDATA%\ODEI\logs\
dir
```

## Troubleshooting

### No logs appearing?

1. Check log directory exists
2. Check `LOG_LEVEL` isn't set to `fatal`
3. Check file permissions

### MCP logs missing?

MCP servers log to **stderr**, not files:
```bash
node dist/index.js 2> server.log
```

### Logs too verbose?

Increase log level:
```bash
export LOG_LEVEL=warn
```

### Performance impact?

Disable debug endpoints:
```bash
export ODEI_DEBUG_ENDPOINTS=0
export LOG_LEVEL=error
```

## Best Practices

✅ **DO:**
- Use appropriate log levels
- Include context data
- Use correlation IDs for distributed operations
- Sanitize sensitive data
- Monitor metrics regularly

❌ **DON'T:**
- Log passwords or tokens
- Use `console.log` in production (use `window.odei.logger`)
- Log in tight loops
- Include large objects without filtering

## Full Documentation

See [/docs/LOGGING.md](/docs/LOGGING.md) for complete documentation.
