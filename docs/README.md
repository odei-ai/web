# ODEI Documentation

Comprehensive documentation for the ODEI project architecture, APIs, and guidelines.

## Quick Links

### Architecture Decision Records (ADRs)

ADRs document major architectural decisions and their rationale.

📁 **[ADR Index](./adr/README.md)** - Complete list of architecture decisions

Key ADRs:

- [ADR-001: Singleton Pattern](./adr/001-singleton-pattern.md) - Why Store, FreezeDetector, and mcpClient are singletons
- [ADR-002: Freeze Detection Architecture](./adr/002-freeze-detection.md) - Web Worker-based main thread monitoring
- [ADR-003: MCP Server Architecture](./adr/003-mcp-server-architecture.md) - Semaphore-based concurrency control
- [ADR-004: State Management](./adr/004-state-management.md) - Centralized state with pub-sub pattern

### API Reference

Complete API documentation for core modules.

- **[Store API](./api/Store.md)** - Centralized state management
- **[mcpClient API](./api/mcpClient.md)** - MCP server communication
- **[UIManager API](./api/UIManager.md)** - UI layout and agent switching

### Guidelines

- **[Performance Guidelines](./PERFORMANCE.md)** - Memory leak prevention, frame budgets, optimization patterns

### Design System

- **[Premium Dark Theme](./premium-dark-theme.md)** - UI color palette, typography, button hierarchy

## Documentation Structure

```
docs/
├── README.md                      # This file - documentation index
├── PERFORMANCE.md                 # Performance guidelines and budgets
├── premium-dark-theme.md         # UI design system specification
│
├── adr/                          # Architecture Decision Records
│   ├── README.md                 # ADR index
│   ├── 000-template.md           # Template for new ADRs
│   ├── 001-singleton-pattern.md
│   ├── 002-freeze-detection.md
│   ├── 003-mcp-server-architecture.md
│   └── 004-state-management.md
│
├── api/                          # API Reference Documentation
│   ├── Store.md
│   ├── mcpClient.md
│   └── UIManager.md
│
├── architecture/                 # Architecture diagrams and docs
├── manuals/                      # User manuals and guides
└── reports/                      # Technical reports
```

## For New Developers

Start here to understand the codebase:

1. **Architecture Overview**
   - [ADR-001: Singleton Pattern](./adr/001-singleton-pattern.md) - Core service pattern
   - [ADR-004: State Management](./adr/004-state-management.md) - How state flows

2. **Performance Critical**
   - [ADR-002: Freeze Detection](./adr/002-freeze-detection.md) - How we monitor performance
   - [Performance Guidelines](./PERFORMANCE.md) - Memory leak prevention, budgets

3. **API Essentials**
   - [Store API](./api/Store.md) - Global state access
   - [mcpClient API](./api/mcpClient.md) - Calling MCP servers
   - [UIManager API](./api/UIManager.md) - UI management

4. **UI Design**
   - [Premium Dark Theme](./premium-dark-theme.md) - Colors, typography, button styles

## For Contributors

### Creating a New ADR

When making a significant architectural decision:

1. Copy `adr/000-template.md` to `adr/NNN-title.md`
2. Fill in all sections (Context, Decision, Consequences, Alternatives)
3. Set status to "Proposed"
4. Discuss with team
5. Update status to "Accepted" when finalized
6. Add entry to `adr/README.md`

### Updating API Docs

When changing public APIs:

1. Update corresponding API doc in `api/`
2. Add migration notes if breaking change
3. Update examples to show new patterns
4. Cross-reference related ADRs

### Performance Impact

Before merging performance-critical code:

1. Review [Performance Guidelines](./PERFORMANCE.md)
2. Verify memory leak prevention patterns used
3. Check frame budgets (16ms target, 33ms max)
4. Test with FreezeDetector and CriticalDiagnostic
5. Document any new performance considerations

## Key Concepts

### Singleton Pattern

ODEI uses module singletons for core services (Store, FreezeDetector, mcpClient). See [ADR-001](./adr/001-singleton-pattern.md).

```javascript
import { store } from './modules/Store.js';
import { freezeDetector } from './modules/FreezeDetector.js';
import { mcpClient } from './modules/mcpClient.js';
```

### State Management

All application state lives in Store, a lightweight pub-sub system. See [ADR-004](./adr/004-state-management.md).

```javascript
// Subscribe to state changes
const unsub = store.subscribe((state) => {
  updateUI(state);
});

// Update state
store.setActiveModule('memory');
```

### MCP Communication

All MCP server calls go through mcpClient with automatic concurrency limiting, timeouts, and retries. See [ADR-003](./adr/003-mcp-server-architecture.md).

```javascript
const { result, latency } = await mcpClient.callTool('odei-neo4j', 'odei.neo4j.search.v1', { query: 'test' });
```

### Performance Monitoring

FreezeDetector automatically monitors main thread freezes. See [ADR-002](./adr/002-freeze-detection.md).

```javascript
// Track expensive operations
const end = freezeDetector.trackOperation('render-graph', { nodeCount: 500 });
try {
  renderGraph();
} finally {
  end();
}
```

## Common Tasks

### Debug Memory Leak

1. Open DevTools → Memory tab
2. Take heap snapshot
3. Interact with app (e.g., switch agents 5 times)
4. Take another snapshot
5. Compare - look for "Detached DOM tree" or growing objects
6. Check if event listeners were cleaned up:
   ```javascript
   window.odei.uiManager.disposables.length;
   window.odei.uiManager.conversationRowDisposables.length;
   ```

### Debug Performance Freeze

1. Open diagnostic overlay (press `` backtick)
2. Reproduce freeze
3. Check event log for freeze events (red entries)
4. Check console for FreezeDetector diagnostics
5. Review active operations at time of freeze:
   ```javascript
   window.freezeDetector.getLog();
   ```

### Check MCP Performance

```javascript
// Listen to metrics
mcpClient.onMetrics((metric) => {
  console.log(`${metric.toolName}: ${metric.latency}ms - ${metric.ok ? 'OK' : 'FAIL'}`);
});

// Check queue state
window.mcpClient._activeRequests; // Current active
window.mcpClient._requestQueue.length; // Queued
```

### View Current State

```javascript
// In DevTools console
window.odeiStore.getState();
window.odei.uiManager.store.getState().activeModule;
```

## Contributing

When adding new features:

1. **Check existing ADRs** - Don't reinvent patterns
2. **Write ADR for major decisions** - Document "why"
3. **Update API docs** - Keep them current
4. **Follow performance guidelines** - Prevent leaks, respect budgets
5. **Use premium dark theme** - Colors, typography, button styles

## Questions?

- **Architecture**: See [ADR Index](./adr/README.md)
- **API Usage**: See [API Reference](./api/)
- **Performance**: See [Performance Guidelines](./PERFORMANCE.md)
- **UI Design**: See [Premium Dark Theme](./premium-dark-theme.md)
