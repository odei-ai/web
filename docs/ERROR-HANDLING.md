# ODEI Error Handling Guide

**Version:** 1.0
**Last Updated:** 2025-12-25

## Overview

ODEI implements comprehensive error handling across all layers:

- **Renderer Process:** Global error handlers, UI error boundaries
- **Main Process:** IPC error standardization, service error handling
- **MCP Servers:** Standardized error responses
- **UI Components:** Graceful degradation, user-friendly error messages

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Renderer Process                          │
├─────────────────────────────────────────────────────────────────┤
│ error-init.js                                                    │
│  ├─ Global error handlers (uncaught errors, promise rejections) │
│  ├─ Error reporting to main process                             │
│  └─ Crash recovery                                              │
├─────────────────────────────────────────────────────────────────┤
│ modules/ErrorHandler.js                                          │
│  ├─ ErrorCodes: Standardized error codes                        │
│  ├─ ODEIError: Custom error class with metadata                 │
│  ├─ errorHandler: Global error logger and event system          │
│  └─ Utility functions: retry, timeout, error creation           │
├─────────────────────────────────────────────────────────────────┤
│ modules/ui/ErrorComponents.js                                   │
│  ├─ ErrorPanel: Full error display with recovery options        │
│  ├─ ErrorToast: Temporary notifications                         │
│  ├─ ErrorBoundary: Component-level error catching               │
│  └─ Helper functions: showError, showWarning, showInfo          │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                         Main Process                             │
├─────────────────────────────────────────────────────────────────┤
│ electron/ipc/error-responses.js                                 │
│  ├─ createErrorResponse: Standard error format                  │
│  ├─ createSuccessResponse: Standard success format              │
│  ├─ wrapIPCHandler: Error handling wrapper for IPC              │
│  ├─ validateParams: Parameter validation                        │
│  └─ CommonErrors: Predefined error responses                    │
└─────────────────────────────────────────────────────────────────┘
```

## Error Codes

### Network/Connectivity (1xxx)

| Code                  | Value | Description                   |
| --------------------- | ----- | ----------------------------- |
| `NETWORK_TIMEOUT`     | 1001  | Request timed out             |
| `NETWORK_UNAVAILABLE` | 1002  | Network connection lost       |
| `CONNECTION_REFUSED`  | 1003  | Server refused connection     |

### MCP/Service (2xxx)

| Code                      | Value | Description                   |
| ------------------------- | ----- | ----------------------------- |
| `MCP_SERVER_UNAVAILABLE`  | 2001  | MCP server not available      |
| `MCP_TOOL_NOT_FOUND`      | 2002  | Requested tool doesn't exist  |
| `MCP_INVALID_PARAMS`      | 2003  | Invalid tool parameters       |
| `MCP_TIMEOUT`             | 2004  | MCP operation timed out       |
| `MCP_AUTH_FAILED`         | 2005  | Authentication failed         |

### IPC (3xxx)

| Code                    | Value | Description                   |
| ----------------------- | ----- | ----------------------------- |
| `IPC_TIMEOUT`           | 3001  | IPC call timed out            |
| `IPC_INVALID_RESPONSE`  | 3002  | Invalid response from main    |
| `IPC_HANDLER_NOT_FOUND` | 3003  | IPC handler not registered    |

### Data/State (4xxx)

| Code                    | Value | Description                   |
| ----------------------- | ----- | ----------------------------- |
| `DATA_PARSE_ERROR`      | 4001  | Failed to parse data          |
| `DATA_VALIDATION_ERROR` | 4002  | Data validation failed        |
| `DATA_NOT_FOUND`        | 4003  | Requested data not found      |
| `STATE_CORRUPTED`       | 4004  | Application state invalid     |

### UI/Rendering (5xxx)

| Code                      | Value | Description                   |
| ------------------------- | ----- | ----------------------------- |
| `RENDER_FAILED`           | 5001  | Component rendering failed    |
| `MODULE_INIT_FAILED`      | 5002  | Module initialization failed  |
| `COMPONENT_MOUNT_FAILED`  | 5003  | Component mounting failed     |

### Agent/Terminal (6xxx)

| Code                    | Value | Description                   |
| ----------------------- | ----- | ----------------------------- |
| `AGENT_START_FAILED`    | 6001  | Failed to start agent         |
| `AGENT_NOT_RUNNING`     | 6002  | Agent is not running          |
| `TERMINAL_INIT_FAILED`  | 6003  | Terminal initialization failed|

### Generic (9xxx)

| Code            | Value | Description                   |
| --------------- | ----- | ----------------------------- |
| `UNKNOWN_ERROR` | 9999  | Unclassified error            |

## Usage Patterns

### 1. Renderer: Async Operations

```javascript
import { errorHandler, ErrorCodes, ODEIError } from './modules/ErrorHandler.js';
import { showErrorToast } from './modules/ui/ErrorComponents.js';

async function loadData() {
  try {
    const response = await fetch('/api/data');
    if (!response.ok) {
      throw new ODEIError(
        `Failed to load data: ${response.statusText}`,
        ErrorCodes.NETWORK_UNAVAILABLE,
        'error',
        { status: response.status }
      );
    }
    return await response.json();
  } catch (error) {
    errorHandler.logError(error, { operation: 'loadData' });
    showErrorToast(errorHandler.formatUserMessage(error), { duration: 5000 });
    throw error;
  }
}
```

### 2. Renderer: Component Initialization with Error Boundary

```javascript
import { ErrorBoundary } from './modules/ui/ErrorComponents.js';

class MyModule {
  constructor(container) {
    this.container = container;
    this.boundary = new ErrorBoundary(container, {
      name: 'MyModule',
      canRetry: true,
      onRetry: () => this.initialize(),
    });
  }

  async initialize() {
    await this.boundary.render(async () => {
      // Initialization code that might fail
      await this.loadData();
      this.render();
    });
  }
}
```

### 3. Renderer: Error Panel Display

```javascript
import { createErrorPanel } from './modules/ui/ErrorComponents.js';

function showModuleError(error) {
  const container = document.getElementById('module-container');
  container.innerHTML = '';

  const panel = createErrorPanel(error, {
    title: 'Module Failed to Load',
    canRetry: true,
    onRetry: () => location.reload(),
    showDetails: true,
  });

  container.appendChild(panel);
}
```

### 4. Renderer: Toast Notifications

```javascript
import { showError, showWarning, showInfo } from './modules/ui/ErrorComponents.js';

// Error toast
showError('Failed to save changes', { duration: 5000 });

// Warning toast
showWarning('Connection is slow', { duration: 3000 });

// Info toast
showInfo('Data synchronized', { duration: 2000 });
```

### 5. Main Process: IPC Handlers with Error Wrapping

```javascript
const { wrapIPCHandler, CommonErrors, createSuccessResponse } = require('./ipc/error-responses');

ipcMain.handle(
  'my-operation',
  wrapIPCHandler(async (event, params) => {
    // Validate params
    if (!params.id) {
      return CommonErrors.invalidParams('Missing id parameter');
    }

    // Check service availability
    if (!myService.isAvailable()) {
      return CommonErrors.serviceUnavailable('MyService');
    }

    // Perform operation
    const result = await myService.doOperation(params.id);

    return createSuccessResponse(result, { operationId: params.id });
  }, 'my-operation')
);
```

### 6. Main Process: Service Error Handling

```javascript
const { createErrorResponse, ErrorCodes } = require('./ipc/error-responses');

class MyService {
  async performOperation() {
    try {
      // ... operation code
    } catch (error) {
      console.error('[MyService] Operation failed:', error);

      // Map to specific error code
      if (error.code === 'ECONNREFUSED') {
        return createErrorResponse(
          ErrorCodes.SERVICE_UNAVAILABLE,
          'Could not connect to backend service',
          { originalError: error.message }
        );
      }

      // Generic error
      return createErrorResponse(
        ErrorCodes.UNKNOWN,
        error.message,
        { stack: error.stack }
      );
    }
  }
}
```

### 7. Utility: Retry with Backoff

```javascript
import { retryWithBackoff } from './modules/ErrorHandler.js';

const data = await retryWithBackoff(
  async () => {
    const response = await fetch('/api/data');
    if (!response.ok) throw new Error('Failed');
    return response.json();
  },
  3,    // max retries
  1000  // base delay (ms)
);
```

### 8. Utility: Timeout Wrapper

```javascript
import { withTimeout } from './modules/ErrorHandler.js';

const data = await withTimeout(
  fetch('/api/slow-operation'),
  5000, // timeout ms
  'Operation timed out after 5 seconds'
);
```

## IPC Response Format

### Success Response

```json
{
  "success": true,
  "data": { /* actual data */ },
  "meta": {
    "timestamp": "2025-12-25T10:30:00.000Z",
    /* optional metadata */
  }
}
```

### Error Response

```json
{
  "success": false,
  "error": {
    "code": "MCP_SERVER_UNAVAILABLE",
    "message": "odei-neo4j is temporarily unavailable",
    "timestamp": "2025-12-25T10:30:00.000Z",
    "context": {
      "server": "odei-neo4j",
      /* additional context */
    }
  }
}
```

## MCP Server Error Handling

MCP servers should return errors in this format:

```typescript
// Success
{
  content: [{ type: "text", text: "Result data" }],
  isError: false
}

// Error
{
  content: [{
    type: "text",
    text: JSON.stringify({
      error: {
        code: "MCP_TOOL_ERROR",
        message: "User-friendly error message",
        details: "Technical details for debugging"
      }
    })
  }],
  isError: true
}
```

## Error Severity Levels

```javascript
import { ErrorSeverity } from './modules/ErrorHandler.js';

ErrorSeverity.INFO;     // Informational, no user action needed
ErrorSeverity.WARNING;  // Warning, user might want to act
ErrorSeverity.ERROR;    // Error, user should be notified
ErrorSeverity.CRITICAL; // Critical, requires immediate attention
```

## User-Friendly Messages

Error codes are automatically mapped to user-friendly messages:

| Error Code              | User Message                                   |
| ----------------------- | ---------------------------------------------- |
| `NETWORK_TIMEOUT`       | "Request timed out. Please try again."         |
| `MCP_SERVER_UNAVAILABLE`| "Service is temporarily unavailable."          |
| `STATE_CORRUPTED`       | "Application state corrupted. Please reload."  |
| `AGENT_START_FAILED`    | "Failed to start agent."                       |

Override with custom message:

```javascript
const error = new ODEIError(
  'Custom user-friendly message',
  ErrorCodes.MCP_TIMEOUT,
  ErrorSeverity.ERROR
);
```

## Recovery Suggestions

Automatically provided for common errors:

```javascript
const suggestion = errorHandler.getRecoverySuggestion(error);
// Returns: "Try again or check your internet connection."
```

## Error Logging

All errors are automatically logged with context:

```javascript
errorHandler.logError(error, {
  module: 'TerminalManager',
  action: 'initialization',
  userId: 'anton',
});
```

View error log:

```javascript
const log = errorHandler.getErrorLog();
console.log(log);
```

Export error log:

```javascript
const jsonLog = errorHandler.exportErrorLog();
// Download or send to logging service
```

## Global Error Handlers

Automatically initialized in `error-init.js`:

1. **Uncaught errors** - Caught and displayed as toast
2. **Unhandled promise rejections** - Caught and displayed as toast
3. **Console.error interceptor** - Logged to error handler
4. **Crash recovery** - Detects previous crashes, offers log export

## Testing Error Handling

### Development Mode

```javascript
// In browser console
import { testErrorHandling } from './error-init.js';
testErrorHandling(); // Triggers test errors
```

### Production Testing

Test error paths by simulating failures:

1. **Network timeout:** Disable network, try MCP call
2. **Service unavailable:** Stop Neo4j, try data fetch
3. **Invalid data:** Corrupt localStorage, reload
4. **Permission denied:** Deny clipboard access

## Best Practices

### DO

- ✅ Always wrap async operations in try/catch
- ✅ Use specific error codes instead of generic
- ✅ Provide context when logging errors
- ✅ Show user-friendly messages, not stack traces
- ✅ Offer recovery options (retry, reload)
- ✅ Log errors to main process for critical issues
- ✅ Use ErrorBoundary for component-level errors
- ✅ Test error paths regularly

### DON'T

- ❌ Fail silently (always log)
- ❌ Show technical errors to users
- ❌ Leak sensitive info in error messages
- ❌ Block UI with error modals
- ❌ Retry infinitely without backoff
- ❌ Ignore promise rejections
- ❌ Use generic "Unknown error" messages

## UI Components Reference

### ErrorPanel

Full error display with recovery options.

**Props:**
- `title`: Error title (default: "Error")
- `canRetry`: Show retry button (default: false)
- `onRetry`: Retry callback
- `canDismiss`: Show dismiss button (default: true)
- `onDismiss`: Dismiss callback
- `showDetails`: Show technical details (default: true)

### ErrorToast

Temporary notification.

**Props:**
- `message`: Error message
- `duration`: Auto-dismiss time in ms (default: 5000)
- `severity`: 'info' | 'warning' | 'error' | 'critical' (default: 'error')
- `canDismiss`: Show close button (default: true)

### ErrorBoundary

Component-level error catcher.

**Constructor:**
- `container`: DOM element
- `options`:
  - `name`: Component name
  - `showError`: Show error UI (default: true)
  - `canRetry`: Show retry button (default: true)
  - `onError`: Error callback
  - `onRetry`: Retry callback

## Debugging

### Enable verbose error logging

```javascript
// In renderer console
window.ODEI_DEBUG_ERRORS = true;

// Will log full error objects
errorHandler.subscribe((error, context) => {
  if (window.ODEI_DEBUG_ERRORS) {
    console.log('Full Error:', error);
    console.log('Context:', context);
  }
});
```

### Export error log for analysis

```javascript
// In renderer console
const log = errorHandler.exportErrorLog();
console.log(log); // Copy to file
```

### Monitor error frequency

```javascript
// In renderer console
const errors = errorHandler.getErrorLog();
console.log(`Total errors: ${errors.length}`);
console.log('Recent errors:', errors.slice(-10));
```

## Future Enhancements

1. **Error telemetry** - Send anonymized errors to logging service
2. **Error patterns** - Detect recurring error patterns
3. **Auto-recovery** - Automatic retry with exponential backoff
4. **Error dashboards** - Visual error analytics in UI
5. **Sentry integration** - Professional error tracking

## Support

For error handling questions:

1. Check this documentation
2. Review `src/modules/ErrorHandlingPatterns.md`
3. Search error code in codebase
4. Check console for detailed logs

---

**Remember:** Good error handling is invisible when things work, invaluable when they don't.
