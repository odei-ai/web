# ADR-001: Singleton Pattern for Core Services

**Date**: 2025-12-13
**Status**: Accepted
**Deciders**: AI Principal, Infrastructure Developer
**Tags**: architecture, patterns, state-management, performance

## Context

ODEI's frontend requires several core services that need to be:

- **Globally accessible** across the application (no prop drilling through 10+ component layers)
- **Single source of truth** (one `Store`, one `FreezeDetector`, one `mcpClient`)
- **Initialized early** in the application lifecycle
- **Shared across modules** (renderer.js, UIManager.js, TerminalManager.js, etc.)
- **Memory-efficient** (no duplicate instances causing resource leaks)

The application has three critical services:

1. **Store** (`src/modules/Store.js`) - Centralized state management
2. **FreezeDetector** (`src/modules/FreezeDetector.js`) - Performance monitoring
3. **MCPClient** (`src/modules/mcpClient.js`) - MCP server communication with concurrency control

These services are imported by 10+ modules and need coordinated initialization/cleanup.

### Forces at Play

- **Global state access**: Multiple disconnected modules need access to shared state
- **Initialization order**: Services must be created before use but after DOM is ready
- **Memory leaks**: Multiple instances would duplicate event listeners, timers, workers
- **Testability**: Singleton makes unit testing harder (but we prioritize runtime stability)
- **Module coupling**: Singletons create implicit dependencies between modules

## Decision

**Use the Module Singleton pattern** (not Class Singleton) for `Store`, `FreezeDetector`, and `MCPClient`.

### Implementation Approach

**Export a single instance** created at module load time:

```javascript
// Store.js
class Store {
  constructor() {
    /* ... */
  }
  // ...
}

// Export singleton instance
export const store = new Store();

// Expose to window for legacy/debugging
if (typeof window !== 'undefined') {
  window.odeiStore = store;
}
```

**Auto-initialize** services that don't need configuration:

```javascript
// FreezeDetector.js
export const freezeDetector = new FreezeDetector();
freezeDetector.start(); // Auto-start on import
```

**Manual configuration** for services with options:

```javascript
// mcpClient.js
export const mcpClient = new MCPClient({
  toolTimeoutMs: 10000,
  maxConcurrent: 6,
});
```

### Why This Approach

1. **Simple** - No getInstance() boilerplate, just import and use
2. **Explicit** - The singleton is visible in the code (`export const store`)
3. **Tree-shakeable** - Unused singletons won't be bundled
4. **Debuggable** - Instance available on `window` for console inspection
5. **ES6 native** - Uses module system, no hacks or private constructors

## Consequences

### Positive Consequences

- **Zero memory overhead** from duplicate instances
- **Consistent state** across all modules
- **Simple API** - `import { store } from './Store.js'` just works
- **Early failure** - Module load errors happen at startup, not mid-session
- **Performance** - No getInstance() function call overhead
- **Debugging** - `window.odeiStore` accessible in DevTools console

### Negative Consequences

- **Testing complexity** - Hard to reset singleton state between tests
- **Tight coupling** - Modules that import singleton can't easily use mocks
- **Hidden dependencies** - Not obvious from function signatures what depends on what
- **Initialization control** - Can't defer creation or create multiple test instances
- **Circular dependencies** - If singletons import each other, can cause issues

### Neutral Consequences

- **Module scope** - Singleton lives in module scope, not class private
- **Window exposure** - Debugging convenience but also global namespace pollution
- **Auto-initialization** - Services start immediately on first import (good for monitoring, tricky for testing)

## Alternatives Considered

### Alternative 1: Dependency Injection (DI)

**Description**: Pass `store`, `freezeDetector`, `mcpClient` as constructor parameters to every class that needs them.

```javascript
class UIManager {
  constructor({ store, freezeDetector, mcpClient, terminals, ... }) {
    this.store = store;
    this.freezeDetector = freezeDetector;
    // ...
  }
}
```

**Pros**:

- Explicit dependencies in constructor
- Easy to mock for testing
- Clear ownership and lifecycle

**Cons**:

- **Verbose** - Every class constructor becomes parameter soup
- **Prop drilling** - Need to pass services through 5+ layers
- **Initialization complexity** - Manual wiring in renderer.js
- **No type safety** - Plain objects, no IDE autocomplete

**Why rejected**: Too much boilerplate for a 40+ module app. The testability gain doesn't justify the development friction.

### Alternative 2: Class Singleton with `getInstance()`

**Description**: Traditional singleton with private constructor and static method.

```javascript
class Store {
  static #instance = null;

  constructor() {
    if (Store.#instance) {
      throw new Error('Use Store.getInstance()');
    }
    Store.#instance = this;
  }

  static getInstance() {
    if (!Store.#instance) {
      Store.#instance = new Store();
    }
    return Store.#instance;
  }
}
```

**Pros**:

- Lazy initialization (only created when first accessed)
- Enforces singleton (constructor throws if called directly)
- Familiar pattern from OOP languages

**Cons**:

- **Verbose API** - `Store.getInstance()` vs `store`
- **No real benefit** - We want eager initialization anyway
- **Extra indirection** - Function call on every access
- **Class-based** - Doesn't leverage ES6 modules

**Why rejected**: Module singletons are simpler and sufficient. The enforced-singleton guarantee isn't worth the API complexity.

### Alternative 3: Service Locator / Registry

**Description**: Global registry that holds all services.

```javascript
// services.js
const services = new Map();

export function registerService(name, instance) {
  services.set(name, instance);
}

export function getService(name) {
  return services.get(name);
}

// Usage
registerService('store', new Store());
const store = getService('store');
```

**Pros**:

- Central registration point
- Can swap implementations at runtime
- Decouples service consumers from concrete types

**Cons**:

- **Stringly-typed** - `getService('store')` has no type safety
- **Indirect** - Harder to trace where services come from
- **Runtime errors** - Typos in service names fail at runtime
- **Overkill** - We only have 3 singletons, not 50

**Why rejected**: Too much indirection for small benefit. String-based lookup is error-prone.

## Implementation Notes

### Key Files

- `/src/modules/Store.js` - Centralized state management singleton
- `/src/modules/FreezeDetector.js` - Performance monitoring singleton
- `/src/modules/mcpClient.js` - MCP communication singleton
- `/src/renderer.js` - Imports and wires up singletons

### Usage Pattern

```javascript
// Correct usage
import { store } from './modules/Store.js';
import { freezeDetector } from './modules/FreezeDetector.js';
import { mcpClient } from './modules/mcpClient.js';

// Store state
store.setActiveModule('memory');
const state = store.getState();

// Track operation for freeze detection
const endTracking = freezeDetector.trackOperation('my-op', { data: 'meta' });
// ... do work ...
endTracking();

// Call MCP tool with semaphore
const { result } = await mcpClient.callTool('odei-neo4j', 'tool.name', params);
```

### Testing Considerations

Since singletons are hard to reset, prefer **integration tests** over unit tests for modules that depend on them.

For unit tests that must mock singletons:

1. Use `jest.mock()` or similar to replace module exports
2. Restore original after each test
3. Avoid tests that mutate global state

### Migration Path

No migration needed - this is the current pattern. Document it as official architecture.

## References

- `/src/modules/Store.js` - Implementation
- `/src/modules/FreezeDetector.js` - Implementation
- `/src/modules/mcpClient.js` - Implementation
- [JavaScript Module Singletons](https://stackoverflow.com/questions/1479319/simplest-cleanest-way-to-implement-singleton-in-javascript)
- ADR-004: State Management - Explains why Store is needed
