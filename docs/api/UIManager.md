# UIManager API Reference

**Module**: `src/modules/UIManager.js`
**Type**: Class (single instance)
**Purpose**: Manages UI layout, agent view switching, sidebar rendering, and user interactions

## Overview

The `UIManager` class is the central UI controller for ODEI. It handles:

- Agent tab switching (discuss, plan, execute, memory, etc.)
- Sidebar content rendering (recent conversations, health dashboard)
- Collapsible panel state management
- Agent status indicators (running/stopped)
- Event listener lifecycle management

## Constructor

```javascript
new UIManager(dependencies: UIManagerDependencies)
```

### Dependencies Object

```typescript
interface UIManagerDependencies {
  terminals: Record<string, Terminal>; // Terminal instances by agent
  conversationManager: ConversationManager; // Conversation persistence
  contextUsageTracker: ContextUsageTracker; // Token usage tracking
  getMemoryController: () => MemoryController; // Lazy accessor for memory view
}
```

**Example:**

```javascript
const uiManager = new UIManager({
  terminals: { discuss: discussTerm, plan: planTerm, ... },
  conversationManager: conversationManager,
  contextUsageTracker: contextUsageTracker,
  getMemoryController: () => memoryController
});
```

## Key Methods

### init()

Initialize UI manager - setup all event listeners and components.

**Signature:**

```javascript
init(): void
```

**Behavior:**

- Sets up event listeners for tabs, buttons, search, filters
- Initializes collapsible panels
- Subscribes to Store for state changes
- Sets up privacy HUD, resizer, window events

**Example:**

```javascript
const uiManager = new UIManager(dependencies);
uiManager.init();
```

**Notes:**

- Call once during app initialization
- Must call before user interaction
- Event listeners stored in `this.disposables` for cleanup

---

### switchAgent()

Switch to a different agent view.

**Signature:**

```javascript
switchAgent(agentName: string): void
```

**Parameters:**

- `agentName` - Agent identifier ('today', 'memory', 'discuss', 'plan', etc.)

**Behavior:**

1. Validates agent name (falls back to 'today' if invalid)
2. Calls `onHide()` on previous view (e.g., Dashboard)
3. Hides all terminal containers and deactivates all tabs
4. Shows target terminal and activates target tab
5. Updates sidebar visibility based on view requirements
6. Fits terminal to viewport
7. Updates Store state (`setActiveModule`)
8. Updates sidebar content for new agent
9. Updates recent conversations list
10. Announces view change to screen readers

**Example:**

```javascript
uiManager.switchAgent('memory'); // Switch to Memory Atlas
uiManager.switchAgent('discuss'); // Switch to Discuss agent
```

**View-Specific Behavior:**

- **memory, today** - Hides sidebar (fullscreen canvas)
- **health stack** (dashboard, apple, garmin, mind) - Hides sidebar
- **work agents** (discuss, plan, execute, builder) - Shows sidebar + conversations

**Notes:**

- Robust error handling - falls back to previous view on failure
- Announces to screen reader for accessibility
- Updates URL hash (not implemented yet, but planned)

---

### updateAgentStatus()

Update agent status indicator (running/stopped/error).

**Signature:**

```javascript
updateAgentStatus(agent: string, status?: 'running' | 'stopped' | 'error'): void
```

**Parameters:**

- `agent` - Agent name
- `status` - Status string (default: 'running')

**Behavior:**

- Updates tab status dot (green for running, red for error, gray for stopped)
- Updates footer status indicator
- Updates agent row `data-running` attribute for play/stop button state
- Updates Store state

**Example:**

```javascript
uiManager.updateAgentStatus('discuss', 'running');
uiManager.updateAgentStatus('plan', 'stopped');
uiManager.updateAgentStatus('execute', 'error');
```

---

### updateRecentConversationsUI()

Update the recent conversations UI list.

**Signature:**

```javascript
async updateRecentConversationsUI(filterAgent?: string | null): Promise<void>
```

**Parameters:**

- `filterAgent` - Filter by agent name, or null for all (optional)

**Behavior:**

1. Fetches recent conversations from ConversationManager
2. Filters by search term (if provided)
3. Groups conversations by date (Today, Yesterday, Previous 7 Days, Older)
4. Renders conversation rows with:
   - Agent icon with gradient
   - Preview text
   - Message count badge
   - Timestamp
   - Copy and delete action buttons
5. Wires up click handlers to load conversations
6. **CRITICAL**: Cleans up old event listeners before adding new ones (prevents memory leaks)

**Example:**

```javascript
// Show all conversations
await uiManager.updateRecentConversationsUI();

// Filter by agent
await uiManager.updateRecentConversationsUI('discuss');
```

**Notes:**

- Fires on every agent switch
- Listens to search input for filtering
- Cleans up `conversationRowDisposables` on re-render to prevent listener accumulation

---

### updateModuleSidebar()

Update sidebar content for active agent.

**Signature:**

```javascript
async updateModuleSidebar(module: string): Promise<void>
```

**Parameters:**

- `module` - Agent name

**Behavior:**

- Highlights active agent in status rows
- Renders agent-specific sidebar content:
  - **discuss** - Context usage widget
  - **plan** - Decision metrics, pending plans
  - **execute** - Today's tasks, work sessions
  - **mind** - Recent insights, pattern stats
  - **health/body** - Body sidebar (imported from agents/health)
  - **finance** - Quick actions, market status
- Mounts context usage widgets if available

**Example:**

```javascript
await uiManager.updateModuleSidebar('plan');
```

**Notes:**

- Calls MCP tools via `mcpClient` (semaphore-controlled)
- Caches results for performance (planned, not implemented)
- Fire-and-forget on errors (won't crash app)

---

### destroy()

Cleanup all event listeners and subscriptions.

**Signature:**

```javascript
destroy(): void
```

**Behavior:**

- Calls all cleanup functions in `this.disposables`
- Calls all cleanup functions in `this.conversationRowDisposables`
- Clears both arrays
- Catches and logs cleanup errors

**Example:**

```javascript
uiManager.destroy();
```

**Notes:**

- Call before destroying UIManager instance
- Prevents memory leaks from lingering event listeners
- Safe to call multiple times (idempotent)

## Properties

### Public Properties

```javascript
this.store; // Store singleton
this.terminals; // Terminal instances by agent
this.conversationManager; // Conversation manager
this.contextUsageTracker; // Context tracker
this.electronAPI; // Electron IPC bridge
this.commandView; // Dashboard command view
this.getMemoryController; // Lazy memory controller accessor
```

### Internal State

```javascript
this.sidebarCache; // Cached sidebar content (not used yet)
this.disposables; // Cleanup functions for global listeners
this.conversationRowDisposables; // Cleanup functions for conversation row listeners
this.workGroupOpen; // Work group collapse state
this.healthGroupOpen; // Health group collapse state
this.healthGroupJustOpened; // Flag for health group just opened
this.storeUnsub; // Store unsubscribe function
```

## Event Listeners

UIManager sets up extensive event listeners. All are stored in `this.disposables` for cleanup.

### Global Listeners

- `window.resize` - Debounced terminal resize
- `document.visibilitychange` - Terminal refit on tab focus
- `document.paste` - Global paste handler for images

### UI Interaction Listeners

- Agent tab clicks - Switch agent view
- Agent status row clicks - Switch to agent + control (play/stop)
- Agent control buttons - Start/stop agents
- Work group toggle - Expand/collapse work agents
- Health group toggle - Expand/collapse health agents
- Dashboard group toggle - Expand/collapse dashboard children
- Collapsible panel toggles - Health dashboard, conversations
- Save conversation button - Persist current terminal
- Search/filter inputs - Update recent conversations
- Filter pills - Quick filter by agent

### Lifecycle

```javascript
// Setup (called by init())
this.setupEventListeners();

// Cleanup (called by destroy())
this.disposables.forEach((fn) => fn());
this.disposables = [];
```

## Usage Patterns

### Initialization

```javascript
// In renderer.js
const uiManager = new UIManager({
  terminals: terminals,
  conversationManager: conversationManager,
  contextUsageTracker: contextUsageTracker,
  getMemoryController: () => memoryController,
});

uiManager.init();

// Expose globally for debugging
window.odei = window.odei || {};
window.odei.uiManager = uiManager;
```

### Agent Switching

```javascript
// User clicks "Discuss" tab
uiManager.switchAgent('discuss');

// Programmatic switch
if (condition) {
  uiManager.switchAgent('memory');
}
```

### Status Updates

```javascript
// Listen for agent state changes from main process
window.odei.ipc.on('agent:state-change', (data) => {
  uiManager.updateAgentStatus(data.agent, data.running ? 'running' : 'stopped');
});
```

### Sidebar Updates

```javascript
// After switching agent, sidebar auto-updates
// But can manually refresh:
await uiManager.updateModuleSidebar('plan');
```

## Memory Leak Prevention

UIManager is prone to memory leaks if not properly managed:

### Problem: Conversation Row Listeners

**Before (LEAKED):**

```javascript
updateRecentConversationsUI() {
  // Renders new conversation rows
  rows.forEach(row => {
    row.addEventListener('click', handler); // LEAK: never removed!
  });
}
```

**After (FIXED):**

```javascript
updateRecentConversationsUI() {
  // CRITICAL: Clean up old listeners first
  this.conversationRowDisposables.forEach(fn => fn());
  this.conversationRowDisposables = [];

  rows.forEach(row => {
    row.addEventListener('click', handler);
    // Store cleanup function
    this.conversationRowDisposables.push(
      () => row.removeEventListener('click', handler)
    );
  });
}
```

### Problem: Document Listeners

**Solution: Store in disposables**

```javascript
const resizeHandler = debounce(() => this.resize(), 100);
window.addEventListener('resize', resizeHandler);
this.disposables.push(() => window.removeEventListener('resize', resizeHandler));
```

## Performance Considerations

### Debounced Operations

- **Window resize** - 100ms debounce
- **Search input** - 300ms debounce
- Terminal fit operations - RAF-debounced via `rafDebounce()`

### Fire-and-Forget Updates

Sidebar updates don't block agent switching:

```javascript
switchAgent(agent) {
  // Synchronous UI updates
  this.showTerminal(agent);
  this.activateTab(agent);

  // Async sidebar updates (don't await)
  this.updateModuleSidebar(agent).catch(err => console.warn(err));
}
```

### Caching (Planned)

```javascript
this.sidebarCache = {}; // Not implemented yet
// Planned: Cache MCP results for 30 seconds to avoid redundant calls
```

## Best Practices

### DO

- Store all event listeners in `this.disposables`
- Clean up conversation row listeners on re-render
- Use debounce for frequent events (resize, input)
- Handle errors gracefully (don't crash on sidebar failure)
- Call `destroy()` before removing UIManager

### DON'T

- Add listeners without cleanup function
- Block agent switching waiting for async operations
- Mutate DOM directly (use methods)
- Call MCP tools synchronously without error handling

## Debugging

### Check Active Agent

```javascript
window.odei.uiManager.store.getState().activeModule;
```

### Check Listener Count

```javascript
window.odei.uiManager.disposables.length;
window.odei.uiManager.conversationRowDisposables.length;
```

### Force Sidebar Update

```javascript
await window.odei.uiManager.updateModuleSidebar('plan');
```

### Check Health Group State

```javascript
window.odei.uiManager.healthGroupOpen;
window.odei.uiManager.workGroupOpen;
```

## Related

- [Store API](./Store.md) - State management
- [mcpClient API](./mcpClient.md) - MCP server calls
- [ADR-004: State Management](../adr/004-state-management.md) - Store integration
- [PERFORMANCE.md](../PERFORMANCE.md) - Memory leak prevention
