# Error Handling Quick Start

Fast reference for implementing error handling in ODEI code.

## 5-Minute Guide

### 1. Async Functions - Always wrap in try/catch

```javascript
import { errorHandler } from './modules/ErrorHandler.js';
import { showErrorToast } from './modules/ui/ErrorComponents.js';

async function myFunction() {
  try {
    const result = await someAsyncOperation();
    return result;
  } catch (error) {
    // Log error
    errorHandler.logError(error, { context: 'myFunction' });

    // Show user notification
    showErrorToast(errorHandler.formatUserMessage(error));

    // Re-throw or return error response
    throw error;
  }
}
```

### 2. Component Initialization - Use ErrorBoundary

```javascript
import { ErrorBoundary } from './modules/ui/ErrorComponents.js';

class MyComponent {
  constructor(container) {
    this.boundary = new ErrorBoundary(container, {
      name: 'MyComponent',
      canRetry: true,
      onRetry: () => this.initialize(),
    });
  }

  async initialize() {
    await this.boundary.render(async () => {
      // Your initialization code
      await this.loadData();
      this.render();
    });
  }
}
```

### 3. Show Error to User

```javascript
import { showError, showWarning, showInfo } from './modules/ui/ErrorComponents.js';

// Error notification
showError('Operation failed');

// Warning notification
showWarning('Connection is slow');

// Info notification
showInfo('Data saved');
```

### 4. IPC Handlers - Wrap with error handling

```javascript
const { wrapIPCHandler, CommonErrors } = require('./ipc/error-responses');

ipcMain.handle('my-operation', wrapIPCHandler(async (event, params) => {
  if (!service.isAvailable()) {
    return CommonErrors.serviceUnavailable('MyService');
  }

  const result = await service.doWork(params);
  return createSuccessResponse(result);
}, 'my-operation'));
```

### 5. Custom Errors - Use ODEIError

```javascript
import { ODEIError, ErrorCodes, ErrorSeverity } from './modules/ErrorHandler.js';

throw new ODEIError(
  'User-friendly message',
  ErrorCodes.MCP_SERVER_UNAVAILABLE,
  ErrorSeverity.ERROR,
  { server: 'odei-neo4j' }
);
```

## Common Patterns

### Pattern: Retry with Backoff

```javascript
import { retryWithBackoff } from './modules/ErrorHandler.js';

const data = await retryWithBackoff(
  async () => await fetch('/api/data'),
  3,    // max retries
  1000  // base delay ms
);
```

### Pattern: Timeout Wrapper

```javascript
import { withTimeout } from './modules/ErrorHandler.js';

const data = await withTimeout(
  slowOperation(),
  5000, // timeout ms
  'Operation timed out'
);
```

### Pattern: Graceful Degradation

```javascript
async function loadOptionalFeature() {
  try {
    const feature = await loadFeature();
    this.feature = feature;
  } catch (error) {
    console.warn('[App] Feature unavailable, using fallback');
    errorHandler.logError(error, { optional: true });
    this.feature = createFallback();
  }
}
```

### Pattern: Error Panel in Container

```javascript
import { createErrorPanel } from './modules/ui/ErrorComponents.js';

function showError(container, error) {
  const panel = createErrorPanel(error, {
    title: 'Failed to Load',
    canRetry: true,
    onRetry: () => retry(),
  });

  container.innerHTML = '';
  container.appendChild(panel);
}
```

## Error Codes Reference

| Code                      | When to Use                          |
| ------------------------- | ------------------------------------ |
| `NETWORK_TIMEOUT`         | Network request timed out            |
| `MCP_SERVER_UNAVAILABLE`  | MCP server not responding            |
| `IPC_TIMEOUT`             | IPC call timed out                   |
| `DATA_PARSE_ERROR`        | Failed to parse JSON/data            |
| `MODULE_INIT_FAILED`      | Component initialization failed      |
| `AGENT_NOT_RUNNING`       | Agent process not running            |

See `docs/ERROR-HANDLING.md` for complete list.

## Checklist for New Code

- [ ] All async functions wrapped in try/catch
- [ ] User-facing errors show friendly messages
- [ ] Errors logged with context
- [ ] Optional features handle errors gracefully
- [ ] IPC handlers return standardized responses
- [ ] Components use ErrorBoundary
- [ ] Recovery options provided (retry, reload)
- [ ] No silent failures

## Testing

Test error paths:

```javascript
// Trigger test error
throw new Error('Test error');

// Test unhandled rejection
Promise.reject(new Error('Test rejection'));

// Test error UI
import { testErrorHandling } from './error-init.js';
testErrorHandling();
```

## Getting Help

1. Read full docs: `docs/ERROR-HANDLING.md`
2. Check examples: `src/modules/ErrorBoundaryExample.js`
3. Review patterns: `src/modules/ErrorHandlingPatterns.md`
4. Search codebase for existing error handling

---

**Remember:** Every error is an opportunity to improve UX. Handle errors gracefully!
