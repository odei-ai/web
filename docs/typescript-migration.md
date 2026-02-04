# TypeScript Migration Guide

## Overview

Critical renderer modules have been migrated from JavaScript to TypeScript for improved type safety and developer experience.

## Migrated Modules

### Core Modules

1. **Store.ts** (`src/modules/Store.ts`)
   - Centralized state management
   - Fully typed state shape and listeners
   - Type-safe actions and subscriptions

2. **mcpClient.ts** (`src/modules/mcpClient.ts`)
   - MCP (Model Context Protocol) client wrapper
   - Typed request/response interfaces
   - Type-safe concurrency control and metrics

### Type Definitions

**src/types/index.ts** - Comprehensive type definitions including:

- Agent types (AgentName, AgentStatus, AgentConfig)
- MCP types (MCPToolParams, MCPToolResult, MCPError, MCPMetrics)
- Store types (StoreState, StoreListener, HealthStatus, Event)
- Conversation types (ConversationMessage, Conversation)
- UI types (TerminalOptions, ContextUsageTracker, ClipboardImage)
- Graph types (GraphNode, GraphEdge, GraphData)
- Window/Global types (ODEIBridge, ElectronAPI, FreezeDetector)

## Build Process

### Configuration

- **tsconfig.json** - TypeScript configuration at project root
  - Target: ES2022 with DOM libraries
  - Strict type checking enabled
  - Module: ES2022 (native ES modules)
  - Output: Compiled .js files alongside .ts sources in `src/`

### Build Commands

```bash
# Type check only (no emit)
npm run typecheck

# Type check with watch mode
npm run typecheck:watch

# Compile TypeScript to JavaScript
npm run build:ts

# Watch mode for TypeScript compilation
npm run watch:ts
```

### Development Workflow

1. **Edit TypeScript files** (`src/modules/*.ts`)
2. **Compile TypeScript**: Run `npm run build:ts` to generate .js files
3. **Type check**: Run `npm run typecheck` to verify types
4. **Import as .js**: Always import using `.js` extension (browser compatibility)

```javascript
// ✅ Correct - import compiled .js file
import { store } from './modules/Store.js';
import { mcpClient } from './modules/mcpClient.js';

// ❌ Wrong - browsers can't load .ts files
import { store } from './modules/Store.ts';
```

## File Structure

```
src/
├── types/
│   └── index.ts          # Type definitions
├── modules/
│   ├── Store.ts          # TypeScript source
│   ├── Store.js          # Compiled JavaScript
│   ├── Store.d.ts        # Type declarations
│   ├── Store.js.map      # Source map
│   ├── mcpClient.ts      # TypeScript source
│   ├── mcpClient.js      # Compiled JavaScript
│   ├── mcpClient.d.ts    # Type declarations
│   └── mcpClient.js.map  # Source map
```

## Benefits

### Type Safety

- Catch errors at compile time instead of runtime
- IntelliSense and autocomplete in editors
- Refactoring confidence with type-checked references

### Code Quality

- Self-documenting code with type annotations
- Enforced strict null checks and type guards
- Better tooling support (VS Code, WebStorm, etc.)

### Maintainability

- Clear interfaces and contracts
- Easier onboarding for new developers
- Reduced runtime errors

## Migration Strategy

### Gradual Migration

The project uses a gradual migration approach:

1. **Start with core modules**: Store and mcpClient (DONE)
2. **Add type definitions**: Comprehensive types in `src/types/` (DONE)
3. **Migrate additional modules**: UIManager, MemoryViewManager, etc. (TODO)
4. **Enable strict mode incrementally**: Already enabled for migrated modules

### Adding New TypeScript Modules

1. Create `.ts` file in `src/modules/`
2. Import types from `src/types/index.ts`
3. Write fully-typed code with JSDoc comments
4. Run `npm run build:ts` to compile
5. Import using `.js` extension in other files

Example:

```typescript
// src/modules/NewModule.ts
import type { StoreState, AgentName } from '../types/index.js';

export class NewModule {
  private state: StoreState;

  constructor(initialState: StoreState) {
    this.state = initialState;
  }

  updateAgent(agent: AgentName): void {
    // Type-safe implementation
  }
}
```

## Troubleshooting

### "Cannot find module" errors

- Ensure you're importing `.js` files, not `.ts` files
- Run `npm run build:ts` to compile TypeScript
- Check that compiled `.js` files exist alongside `.ts` sources

### Type errors

- Run `npm run typecheck` to see all type errors
- Check `src/types/index.ts` for type definitions
- Ensure imports use correct type paths

### Build errors

- Delete generated files and rebuild: `rm src/modules/*.d.ts src/modules/*.js.map && npm run build:ts`
- Check tsconfig.json configuration
- Verify all imports use `.js` extensions

## Future Work

- Migrate UIManager.js to TypeScript
- Migrate MemoryViewManager.js to TypeScript
- Migrate TodayView.js to TypeScript
- Add stricter linting with ESLint + TypeScript plugin
- Configure CI/CD type checking
- Add pre-commit hooks for type checking

## References

- [TypeScript Handbook](https://www.typescriptlang.org/docs/handbook/intro.html)
- [TypeScript with ES Modules](https://www.typescriptlang.org/docs/handbook/esm-node.html)
- Project tsconfig.json configuration
- Type definitions in `src/types/index.ts`
