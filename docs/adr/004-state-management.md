# ADR-004: Centralized State Management with Store

**Date**: 2025-12-13
**Status**: Accepted
**Deciders**: AI Principal, Infrastructure Developer
**Tags**: architecture, state-management, performance, patterns

## Context

ODEI is a multi-agent Electron application with complex UI state:

- **Active module** - Which agent view is currently shown (discuss, plan, execute, memory, etc.)
- **Agent status** - Running/stopped state for 8 agents
- **Health metrics** - Heart rate, sleep, workouts, nutrition
- **UI state** - Sidebar collapse, filter selections, search terms
- **Handoff state** - Pending agent-to-agent handoffs

State was initially scattered across:

- DOM attributes (`data-running="true"`)
- Global variables (`window.activeAgent`)
- Component-local state (each module tracked its own state)
- Event handlers with inline state checks

This caused issues:

- **Stale state** - UI elements showed different values for same state
- **Race conditions** - Multiple components updating same state
- **No single source of truth** - Hard to debug what state actually is
- **Tight coupling** - Components directly manipulated each other's DOM
- **No reactivity** - Manual DOM updates required when state changed

### Requirements

1. **Single source of truth** - One place to check current state
2. **Reactive updates** - UI updates automatically when state changes
3. **Subscription model** - Components listen for state changes
4. **Minimal API** - Simple get/set, no complex reducers or actions
5. **Performance** - Batched notifications to prevent recursive loops
6. **Debuggability** - State visible in DevTools console

## Decision

**Implement a lightweight, pub-sub state Store** with synchronous updates and deferred notifications.

### Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        Store                                 │
│                  (Single Source of Truth)                    │
│                                                              │
│  state = {                                                   │
│    activeModule: 'today',                                   │
│    agentStatus: { discuss: true, plan: false, ... },       │
│    healthStatus: { heartRate: 72, sleep: 7.5, ... }        │
│  }                                                           │
│                                                              │
│  listeners = Set([listener1, listener2, listener3])         │
└──────────────┬───────────────────────────────────────────────┘
               │
               │ notify() → queueMicrotask
               │
               ▼
┌──────────────────────────────────────────────────────────────┐
│                      Subscribers                              │
│                                                              │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │ UIManager   │  │TerminalMgr  │  │ TodayView   │         │
│  └─────────────┘  └─────────────┘  └─────────────┘         │
│                                                              │
│  Each subscriber receives full state snapshot on change     │
└──────────────────────────────────────────────────────────────┘
```

### Implementation Details

**Store Class** (`src/modules/Store.js`):

```javascript
class Store {
  constructor() {
    this.state = {
      activeModule: 'today',
      agentStatus: {},
      healthStatus: {},
      events: [],
    };
    this.listeners = new Set();

    // Prevent recursive notification loops
    this._notifyScheduled = false;
    this._isNotifying = false;
  }

  // Get current state (read-only copy)
  getState() {
    return this.state;
  }

  // Subscribe to state changes
  subscribe(listener) {
    this.listeners.add(listener);
    return () => this.listeners.delete(listener); // Unsubscribe function
  }

  // Notify all listeners - deferred to prevent recursive loops
  notify() {
    // If already in notify cycle, skip (batched)
    if (this._isNotifying) return;

    // Debounce: schedule notification for next microtask
    if (this._notifyScheduled) return;

    this._notifyScheduled = true;
    queueMicrotask(() => {
      this._notifyScheduled = false;
      this._isNotifying = true;
      try {
        const stateCopy = { ...this.state }; // Shallow copy
        this.listeners.forEach((listener) => {
          try {
            listener(stateCopy);
          } catch (err) {
            console.error('[Store] Listener error:', err);
          }
        });
      } finally {
        this._isNotifying = false;
      }
    });
  }

  // Actions - mutate state and notify
  setActiveModule(moduleName) {
    if (this.state.activeModule !== moduleName) {
      this.state.activeModule = moduleName;
      this.notify();
    }
  }

  setAgentStatus(agent, isRunning) {
    if (this.state.agentStatus[agent] !== isRunning) {
      this.state.agentStatus = {
        ...this.state.agentStatus,
        [agent]: isRunning,
      };
      this.notify();
    }
  }

  setHealthStatus(status) {
    this.state.healthStatus = status;
    this.notify();
  }

  addEvent(event) {
    this.state.events = [...this.state.events, event];
    this.notify();
  }
}

// Export singleton
export const store = new Store();

// Expose to window for debugging
if (typeof window !== 'undefined') {
  window.odeiStore = store;
}
```

### Design Principles

1. **Synchronous mutations** - `setState()` immediately updates state
2. **Deferred notifications** - `notify()` deferred to microtask to batch updates
3. **Immutable updates** - Always create new objects/arrays for nested state
4. **Guarded actions** - Only notify if value actually changed (prevents loops)
5. **Error isolation** - Listener errors don't crash other listeners
6. **Shallow copies** - Pass `{ ...state }` to listeners, not deep clone (performance)

### Usage Pattern

```javascript
import { store } from './modules/Store.js';

// Subscribe to state changes
const unsubscribe = store.subscribe((state) => {
  console.log('State updated:', state);
  updateUI(state);
});

// Update state
store.setActiveModule('memory');
store.setAgentStatus('discuss', true);

// Batch updates (notify only fires once at end)
store.setActiveModule('plan');
store.setAgentStatus('plan', true);
store.setAgentStatus('discuss', false);
// → Single notification after all updates

// Cleanup
unsubscribe();
```

## Consequences

### Positive Consequences

- **Single source of truth** - All state in one place
- **Reactive UI** - Components auto-update when state changes
- **No stale state** - All components see same current state
- **Simple API** - Just `getState()`, `subscribe()`, and action methods
- **Batched notifications** - Multiple updates = one notification
- **Recursive loop prevention** - Won't notify during notification
- **Error resilience** - One listener error doesn't break others
- **Debuggable** - `window.odeiStore.getState()` in DevTools

### Negative Consequences

- **Manual actions** - Must call `store.setActiveModule()` instead of direct mutation
- **No time travel** - Can't undo/redo state changes
- **No middleware** - Can't intercept actions for logging, analytics, etc.
- **Shallow equality** - Nested object updates may not trigger re-render (need immutable patterns)
- **Global state** - All state in one store, can grow large

### Neutral Consequences

- **Not Redux** - Simpler than Redux, but less powerful
- **Synchronous** - State updates happen immediately (good for debugging, bad for async coordination)
- **Shallow copy** - Listeners get `{ ...state }`, not deep clone (faster but less safe)

## Alternatives Considered

### Alternative 1: Redux

**Description**: Use Redux for centralized state with actions, reducers, and middleware.

```javascript
import { createStore } from 'redux';

const reducer = (state = initialState, action) => {
  switch (action.type) {
    case 'SET_ACTIVE_MODULE':
      return { ...state, activeModule: action.payload };
    default:
      return state;
  }
};

const store = createStore(reducer);
store.dispatch({ type: 'SET_ACTIVE_MODULE', payload: 'memory' });
```

**Pros**:

- Industry standard, well-documented
- Time travel debugging (Redux DevTools)
- Middleware support (logging, analytics, async)
- Immutability enforced by convention

**Cons**:

- **Verbose** - Requires actions, action creators, reducers, dispatch
- **Boilerplate** - 10+ lines of code per state update
- **Learning curve** - Team needs to learn Redux concepts
- **Overkill** - We have simple state, don't need Redux power
- **Bundle size** - Adds 20KB to bundle

**Why rejected**: Too much boilerplate for our simple state needs. We don't need time travel or complex middleware.

### Alternative 2: MobX

**Description**: Use MobX for reactive state with observables and automatic tracking.

```javascript
import { observable, action, makeObservable } from 'mobx';

class Store {
  activeModule = 'today';
  agentStatus = {};

  constructor() {
    makeObservable(this, {
      activeModule: observable,
      agentStatus: observable,
      setActiveModule: action,
    });
  }

  setActiveModule(module) {
    this.activeModule = module;
  }
}
```

**Pros**:

- Less boilerplate than Redux
- Automatic dependency tracking
- Fine-grained reactivity (only re-render what changed)
- Simpler mental model (just mutate observables)

**Cons**:

- **Magic** - Automatic tracking can be confusing (when does it re-render?)
- **Bundle size** - Adds 30KB to bundle
- **Decorators** - Requires TypeScript or Babel plugin
- **Learning curve** - Team needs to learn MobX concepts
- **Debugging** - Harder to trace why something re-rendered

**Why rejected**: Too much magic and bundle overhead for simple state. Our custom Store is 50 lines and does everything we need.

### Alternative 3: React Context + useState

**Description**: Use React Context API for state management (if we were using React).

```javascript
const StateContext = React.createContext();

function StateProvider({ children }) {
  const [state, setState] = React.useState(initialState);
  return <StateContext.Provider value={{ state, setState }}>{children}</StateContext.Provider>;
}
```

**Pros**:

- Native React solution, no extra libraries
- Component-scoped subscriptions
- Works with React DevTools

**Cons**:

- **We don't use React** - ODEI is vanilla JS with direct DOM manipulation
- **Component-only** - Can't use outside React tree
- **Re-render entire tree** - Context change re-renders all consumers
- **Prop drilling** - Still need to pass context down through components

**Why rejected**: We don't use React. Even if we did, Context has performance issues for frequently-changing state.

### Alternative 4: Custom Events (EventTarget)

**Description**: Use browser's EventTarget API for pub-sub pattern.

```javascript
const stateManager = new EventTarget();

stateManager.addEventListener('statechange', (e) => {
  console.log('State changed:', e.detail);
});

stateManager.dispatchEvent(
  new CustomEvent('statechange', {
    detail: { activeModule: 'memory' },
  })
);
```

**Pros**:

- Native browser API, no library needed
- Familiar event-driven pattern
- Can use event bubbling/capturing

**Cons**:

- **No state storage** - Just events, need separate state object
- **Weak typing** - Events are just strings, easy to typo
- **No unsubscribe return** - Must store handler reference for removal
- **Verbose** - `dispatchEvent(new CustomEvent(...))` is wordy

**Why rejected**: EventTarget is for DOM events, not state management. Our custom Store is more ergonomic.

## Implementation Notes

### Key Files

- `/src/modules/Store.js` - Store implementation
- `/src/modules/UIManager.js` - Primary Store consumer
- `/src/renderer.js` - Store initialization

### Usage Pattern

**Subscribe to state changes:**

```javascript
import { store } from './modules/Store.js';

class UIManager {
  init() {
    // Subscribe and store unsubscribe function
    this.storeUnsub = store.subscribe((state) => {
      this.updateUI(state);
    });
  }

  updateUI(state) {
    // Update DOM based on state
    if (state.activeModule === 'memory') {
      this.showMemoryView();
    }
  }

  destroy() {
    // Cleanup subscription
    if (this.storeUnsub) this.storeUnsub();
  }
}
```

**Update state:**

```javascript
// Switch active module
store.setActiveModule('memory');

// Update agent status
store.setAgentStatus('discuss', true);

// Batch updates (single notification)
store.setAgentStatus('discuss', false);
store.setAgentStatus('plan', true);
store.setActiveModule('plan');
// → notify() fires once after microtask
```

**Debug in console:**

```javascript
// Check current state
window.odeiStore.getState();

// Watch state changes
window.odeiStore.subscribe((state) => console.log('State:', state));
```

### Performance Considerations

- **Microtask batching** - Multiple updates = one notification
- **Shallow copy** - `{ ...state }` is fast, deep clone is slow
- **Guarded updates** - Only notify if value changed (prevents infinite loops)
- **Error isolation** - One listener crash doesn't break others

### Testing Considerations

Since Store is a singleton, tests should:

1. Reset state before each test
2. Clear all listeners before each test
3. Avoid mutating shared state

```javascript
beforeEach(() => {
  // Reset store to initial state
  store.state = {
    activeModule: 'today',
    agentStatus: {},
    healthStatus: {},
    events: [],
  };

  // Clear all listeners
  store.listeners.clear();
});
```

## References

- `/src/modules/Store.js` - Implementation
- `/src/modules/UIManager.js` - Primary consumer
- ADR-001: Singleton Pattern - Explains why Store is a singleton
- [Pub-Sub Pattern](https://en.wikipedia.org/wiki/Publish%E2%80%93subscribe_pattern)
- [Redux Comparison](https://redux.js.org/understanding/thinking-in-redux/motivation)
