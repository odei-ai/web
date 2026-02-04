# Store API Reference

**Module**: `src/modules/Store.js`
**Type**: Singleton
**Purpose**: Centralized state management with pub-sub pattern

## Overview

The `Store` class provides a lightweight, reactive state management solution for ODEI. It maintains a single source of truth for application state and notifies subscribers when state changes.

## Import

```javascript
import { store } from './modules/Store.js';
```

The `store` singleton is auto-instantiated and ready to use immediately.

## State Structure

```typescript
interface State {
  activeModule: string; // Current agent view ('today', 'memory', 'discuss', etc.)
  agentStatus: Record<string, boolean>; // Agent running status { discuss: true, plan: false }
  healthStatus: object; // Health metrics from Apple Health/Garmin
  events: Array<any>; // Event log
}
```

## Methods

### getState()

Returns the current state.

**Signature:**

```javascript
getState(): State
```

**Returns:**

- `State` - Current state object (read-only reference, do not mutate)

**Example:**

```javascript
const state = store.getState();
console.log('Active module:', state.activeModule);
console.log('Discuss agent running:', state.agentStatus.discuss);
```

**Notes:**

- Returns direct reference to state object - **do not mutate**
- To update state, use action methods (setActiveModule, setAgentStatus, etc.)
- State mutations outside actions will not trigger notifications

---

### subscribe()

Subscribe to state changes. Listener is called whenever state updates.

**Signature:**

```javascript
subscribe(listener: (state: State) => void): () => void
```

**Parameters:**

- `listener` - Function called when state changes, receives state snapshot

**Returns:**

- `Function` - Unsubscribe function (call to remove listener)

**Example:**

```javascript
// Subscribe
const unsubscribe = store.subscribe((state) => {
  console.log('State updated:', state);
  updateUI(state);
});

// Cleanup
unsubscribe();
```

**Notes:**

- Listener receives **shallow copy** of state (`{ ...this.state }`)
- Listener errors are caught and logged, won't crash other listeners
- Notifications are **batched** via `queueMicrotask` - multiple updates = one notification
- Recursive calls to `notify()` are ignored (prevents infinite loops)

---

### notify()

Trigger notifications to all subscribers. **Internal method** - typically not called directly.

**Signature:**

```javascript
notify(): void
```

**Behavior:**

- Defers notification to next microtask using `queueMicrotask`
- Batches multiple calls into single notification
- Prevents recursive notification loops via `_isNotifying` flag
- Calls all subscribers with shallow copy of state

**Example:**

```javascript
// Usually called automatically by action methods
store.setActiveModule('memory'); // Calls notify() internally

// Direct call (rare - only needed for custom state mutations)
store.state.customField = 'value';
store.notify();
```

---

### setActiveModule()

Update the active module (current agent view).

**Signature:**

```javascript
setActiveModule(moduleName: string): void
```

**Parameters:**

- `moduleName` - Module ID ('today', 'memory', 'discuss', 'plan', 'execute', etc.)

**Example:**

```javascript
store.setActiveModule('memory');
store.setActiveModule('discuss');
```

**Notes:**

- Only notifies if value changed (prevents unnecessary updates)
- Valid module names: `'today'`, `'memory'`, `'discuss'`, `'plan'`, `'execute'`, `'builder'`, `'mind'`, `'health'`, `'finance'`

---

### setAgentStatus()

Update an agent's running status.

**Signature:**

```javascript
setAgentStatus(agent: string, isRunning: boolean): void
```

**Parameters:**

- `agent` - Agent name ('discuss', 'plan', 'execute', etc.)
- `isRunning` - `true` if agent is running, `false` if stopped

**Example:**

```javascript
store.setAgentStatus('discuss', true); // Agent started
store.setAgentStatus('plan', false); // Agent stopped
```

**Notes:**

- Only notifies if value changed
- Creates new `agentStatus` object (immutable update pattern)

---

### setHealthStatus()

Update health metrics.

**Signature:**

```javascript
setHealthStatus(status: object): void
```

**Parameters:**

- `status` - Health status object (structure defined by health dashboard)

**Example:**

```javascript
store.setHealthStatus({
  heartRate: 72,
  sleep: { hours: 7.5, quality: 85 },
  steps: 8432,
  calories: 2100,
});
```

**Notes:**

- Always notifies (no change detection)
- Replaces entire health status object

---

### addEvent()

Add an event to the event log.

**Signature:**

```javascript
addEvent(event: any): void
```

**Parameters:**

- `event` - Event object to add (structure arbitrary)

**Example:**

```javascript
store.addEvent({
  type: 'agent-started',
  agent: 'discuss',
  timestamp: Date.now(),
});
```

**Notes:**

- Creates new events array (immutable update pattern)
- Always notifies

## Internal Properties

### state

Current application state. **Do not mutate directly** - use action methods.

```javascript
this.state = {
  activeModule: 'today',
  agentStatus: {},
  healthStatus: {},
  events: [],
};
```

### listeners

Set of subscriber functions.

```javascript
this.listeners = new Set();
```

### \_notifyScheduled

Flag indicating if notification is scheduled for next microtask.

### \_isNotifying

Flag indicating if currently notifying listeners (prevents recursion).

## Usage Patterns

### Basic Subscription

```javascript
import { store } from './modules/Store.js';

class MyComponent {
  constructor() {
    this.unsubscribe = store.subscribe((state) => {
      this.render(state);
    });
  }

  render(state) {
    // Update UI based on state
    console.log('Active module:', state.activeModule);
  }

  destroy() {
    this.unsubscribe();
  }
}
```

### Batched Updates

Multiple state updates result in single notification:

```javascript
// These three updates trigger only one notification
store.setActiveModule('plan');
store.setAgentStatus('plan', true);
store.setAgentStatus('discuss', false);

// Subscribers receive combined state change after microtask
```

### Debugging in Console

```javascript
// Check current state
window.odeiStore.getState();

// Watch state changes
window.odeiStore.subscribe((state) => console.table(state));

// Check listeners
window.odeiStore.listeners.size;
```

## Performance Considerations

- **Batched notifications** - Multiple updates within same tick = one notification
- **Shallow copy** - Subscribers receive `{ ...state }`, not deep clone
- **Guarded updates** - Only notify if value changed (where applicable)
- **Microtask scheduling** - Uses `queueMicrotask` for deferred notification
- **Error isolation** - Listener errors caught, don't break other listeners

## Best Practices

### DO

- Use action methods to update state
- Store unsubscribe function for cleanup
- Check if value changed before updating
- Use immutable update patterns for nested objects

```javascript
// Good - immutable update
store.state.agentStatus = {
  ...store.state.agentStatus,
  discuss: true,
};
```

### DON'T

- Mutate state directly
- Forget to unsubscribe
- Create infinite notification loops
- Deep clone state (performance cost)

```javascript
// Bad - direct mutation, no notification
store.state.activeModule = 'memory';

// Bad - forget to unsubscribe (memory leak)
store.subscribe((state) => updateUI(state));
// (no cleanup)
```

## Testing

### Reset State Between Tests

```javascript
beforeEach(() => {
  store.state = {
    activeModule: 'today',
    agentStatus: {},
    healthStatus: {},
    events: [],
  };
  store.listeners.clear();
});
```

### Mock Notifications

```javascript
test('updates UI when state changes', () => {
  const mockListener = jest.fn();
  const unsub = store.subscribe(mockListener);

  store.setActiveModule('memory');

  // Wait for microtask
  await Promise.resolve();

  expect(mockListener).toHaveBeenCalledWith(
    expect.objectContaining({ activeModule: 'memory' })
  );

  unsub();
});
```

## Related

- [ADR-001: Singleton Pattern](../adr/001-singleton-pattern.md) - Why Store is a singleton
- [ADR-004: State Management](../adr/004-state-management.md) - Store design rationale
- [UIManager API](./UIManager.md) - Primary Store consumer
