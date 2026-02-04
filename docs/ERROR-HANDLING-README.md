# Error Handling System - README

**ODEI Comprehensive Error Handling**
**Version:** 1.0
**Status:** Production Ready

## Quick Links

- **Quick Start:** [ERROR-HANDLING-QUICKSTART.md](./ERROR-HANDLING-QUICKSTART.md)
- **Full Guide:** [ERROR-HANDLING.md](./ERROR-HANDLING.md)
- **Flow Diagrams:** [ERROR-FLOW-DIAGRAM.md](./ERROR-FLOW-DIAGRAM.md)
- **MCP Servers:** [/servers/MCP-ERROR-HANDLING.md](../servers/MCP-ERROR-HANDLING.md)
- **Implementation Summary:** [/ERROR-HANDLING-SUMMARY.md](../ERROR-HANDLING-SUMMARY.md)

## What Is This?

ODEI's error handling system provides:

✅ **Consistent error handling** across all layers (renderer, main, MCP)
✅ **User-friendly error messages** (no scary stack traces)
✅ **Graceful degradation** (continue working when optional features fail)
✅ **Recovery options** (retry, reload, dismiss)
✅ **Comprehensive logging** (context, timestamps, export)
✅ **Theme-consistent UI** (jade/champagne error components)

## For Users

### What You'll See

**Error Toasts** - Small notifications that auto-dismiss:
```
┌────────────────────────────────┐
│ ⚠️  Service temporarily        │
│     unavailable                │  [×]
└────────────────────────────────┘
```

**Error Panels** - Full error display when something needs attention:
```
┌─────────────────────────────────────────┐
│ ⚠️  Module Failed to Load               │
│                                         │
│ Service is temporarily unavailable.     │
│                                         │
│ Suggestion: Wait a moment and retry.   │
│                                         │
│ [Retry]  [Dismiss]  [Reload App]       │
└─────────────────────────────────────────┘
```

**Recovery Options:**
- **Retry** - Try the operation again
- **Dismiss** - Hide the error and continue
- **Reload** - Refresh the entire app

### When Things Go Wrong

1. **Optional feature fails** → Warning toast, app continues
2. **Critical module fails** → Error panel, must retry or reload
3. **App crashes** → Auto-recovery, previous state restored

## For Developers

### 5-Second Integration

```javascript
import { errorHandler } from './modules/ErrorHandler.js';
import { showErrorToast } from './modules/ui/ErrorComponents.js';

try {
  await yourOperation();
} catch (error) {
  errorHandler.logError(error);
  showErrorToast(errorHandler.formatUserMessage(error));
}
```

### Core Components

1. **ErrorHandler** (`/src/modules/ErrorHandler.js`)
   - Central error management
   - Logging, formatting, recovery suggestions

2. **ErrorComponents** (`/src/modules/ui/ErrorComponents.js`)
   - ErrorPanel, ErrorToast, ErrorBoundary
   - Theme-consistent UI

3. **error-init** (`/src/error-init.js`)
   - Global error handlers
   - Crash recovery

4. **IPC Errors** (`/electron/ipc/error-responses.js`)
   - Standardized IPC responses
   - Error wrapping

### Documentation Structure

```
docs/
├── ERROR-HANDLING-README.md          ← You are here
├── ERROR-HANDLING-QUICKSTART.md      ← 5-minute guide
├── ERROR-HANDLING.md                 ← Full reference
└── ERROR-FLOW-DIAGRAM.md             ← Visual guides

servers/
└── MCP-ERROR-HANDLING.md             ← MCP server guide

/
└── ERROR-HANDLING-SUMMARY.md         ← Implementation summary
```

### Getting Started

**Step 1:** Read the Quick Start
```bash
open docs/ERROR-HANDLING-QUICKSTART.md
```

**Step 2:** Review Examples
```bash
open src/modules/ErrorBoundaryExample.js
```

**Step 3:** Integrate with Your Code
- Wrap async functions in try/catch
- Use ErrorBoundary for components
- Use showErrorToast for user feedback

**Step 4:** Test
```javascript
// In browser console
import { testErrorHandling } from './error-init.js';
testErrorHandling();
```

## Key Concepts

### Error Codes

Every error has a code:
```javascript
ErrorCodes.NETWORK_TIMEOUT      // 1001
ErrorCodes.MCP_SERVER_UNAVAILABLE // 2001
ErrorCodes.MODULE_INIT_FAILED   // 5002
```

### Error Severity

Errors have severity levels:
```javascript
ErrorSeverity.INFO      // Informational
ErrorSeverity.WARNING   // Warning
ErrorSeverity.ERROR     // Error
ErrorSeverity.CRITICAL  // Critical
```

### Error Context

Every logged error includes context:
```javascript
errorHandler.logError(error, {
  module: 'TerminalManager',
  operation: 'initialization',
  params: { agent: 'discuss' }
});
```

### Graceful Degradation

Optional features fail gracefully:
```javascript
try {
  await loadFinanceModule();
} catch (error) {
  console.warn('[App] Finance unavailable');
  // App continues without Finance
}
```

## Architecture

```
┌────────────────────────────────────────────┐
│           Renderer Process                  │
├────────────────────────────────────────────┤
│  error-init.js                             │
│  ├─ Global error handlers                  │
│  └─ Crash recovery                         │
├────────────────────────────────────────────┤
│  ErrorHandler.js                           │
│  ├─ Error codes & logging                  │
│  └─ User-friendly messages                 │
├────────────────────────────────────────────┤
│  ErrorComponents.js                        │
│  ├─ ErrorPanel, ErrorToast                 │
│  └─ ErrorBoundary                          │
└────────────────────────────────────────────┘

┌────────────────────────────────────────────┐
│            Main Process                     │
├────────────────────────────────────────────┤
│  error-responses.js                        │
│  ├─ IPC error standardization              │
│  └─ wrapIPCHandler()                       │
└────────────────────────────────────────────┘

┌────────────────────────────────────────────┐
│            MCP Servers                      │
├────────────────────────────────────────────┤
│  Standard error format                     │
│  ├─ MCPError class                         │
│  └─ createErrorResponse()                  │
└────────────────────────────────────────────┘
```

## Examples

### Example 1: Async Function

```javascript
async function loadData() {
  try {
    const data = await fetch('/api/data');
    return data;
  } catch (error) {
    errorHandler.logError(error, { operation: 'loadData' });
    showErrorToast(errorHandler.formatUserMessage(error));
    throw error;
  }
}
```

### Example 2: Component with Error Boundary

```javascript
class Dashboard {
  constructor(container) {
    this.boundary = new ErrorBoundary(container, {
      name: 'Dashboard',
      canRetry: true,
      onRetry: () => this.initialize()
    });
  }

  async initialize() {
    await this.boundary.render(async () => {
      await this.loadData();
      this.render();
    });
  }
}
```

### Example 3: IPC Handler

```javascript
const { wrapIPCHandler, CommonErrors } = require('./ipc/error-responses');

ipcMain.handle('get-data', wrapIPCHandler(async (event, params) => {
  if (!service.isAvailable()) {
    return CommonErrors.serviceUnavailable('DataService');
  }

  const data = await service.getData(params);
  return createSuccessResponse(data);
}, 'get-data'));
```

## Testing

### Manual Testing

1. **Test uncaught error:**
   ```javascript
   throw new Error('Test error');
   ```

2. **Test unhandled rejection:**
   ```javascript
   Promise.reject(new Error('Test rejection'));
   ```

3. **Test error UI:**
   ```javascript
   import { testErrorHandling } from './error-init.js';
   testErrorHandling();
   ```

### Automated Testing

See `docs/ERROR-HANDLING.md` for comprehensive testing guide.

## Best Practices

### ✅ DO

- Always wrap async operations in try/catch
- Use specific error codes
- Provide context when logging errors
- Show user-friendly messages
- Offer recovery options
- Test error paths

### ❌ DON'T

- Fail silently
- Show technical errors to users
- Leak sensitive info in errors
- Block UI indefinitely
- Ignore promise rejections
- Use generic error messages

## Integration Checklist

- [ ] Read Quick Start guide
- [ ] Review examples
- [ ] Add try/catch to async functions
- [ ] Use ErrorBoundary for components
- [ ] Update IPC handlers to use error-responses.js
- [ ] Test error flows
- [ ] Document error handling in your module

## Troubleshooting

### "Error handling not working"
1. Check that error-init.js is loaded
2. Verify imports are correct
3. Check browser console for errors

### "Errors not showing to user"
1. Ensure ErrorComponents are imported
2. Check showErrorToast is called
3. Verify DOM structure for error display

### "IPC errors not standardized"
1. Wrap handlers in wrapIPCHandler()
2. Use createErrorResponse/createSuccessResponse
3. Return standardized format

## Need Help?

1. **Quick questions:** Check ERROR-HANDLING-QUICKSTART.md
2. **Detailed info:** Read ERROR-HANDLING.md
3. **Visual reference:** See ERROR-FLOW-DIAGRAM.md
4. **MCP servers:** Read servers/MCP-ERROR-HANDLING.md
5. **Examples:** Review src/modules/ErrorBoundaryExample.js

## Contributing

When adding new code:

1. Follow error handling patterns
2. Add try/catch to async functions
3. Use ErrorBoundary for components
4. Log errors with context
5. Show user-friendly messages
6. Test error paths

## Version History

**v1.0** (2025-12-25)
- Initial implementation
- Core error handling infrastructure
- UI components
- Global error handlers
- IPC standardization
- Comprehensive documentation

---

**Error handling that works. Errors that inform, not intimidate.**
