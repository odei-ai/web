# ODEI Testing Guide

Comprehensive testing infrastructure for ODEI — Agentic Operating System.

## Table of Contents

- [Overview](#overview)
- [Test Types](#test-types)
- [Running Tests](#running-tests)
- [Writing Tests](#writing-tests)
- [Test Utilities](#test-utilities)
- [Coverage Requirements](#coverage-requirements)
- [CI/CD Integration](#cicd-integration)
- [Troubleshooting](#troubleshooting)

---

## Overview

ODEI uses a multi-layered testing strategy:

- **Unit Tests** — Vitest for isolated component/module testing
- **Integration Tests** — Vitest with real services (Neo4j, MCP servers)
- **E2E Tests** — Playwright for full Electron app testing

### Test Infrastructure

```
tests/
├── unit/               # Vitest unit tests
│   ├── Store.spec.ts
│   ├── FreezeDetector.spec.ts
│   └── helpers/
│       └── testUtils.ts
├── integration/        # Integration tests (to be added)
│   └── neo4j.spec.ts
├── fixtures/           # Shared test data
│   ├── graph-data.ts
│   └── agent-data.ts
├── mocks/             # Mock implementations
│   ├── neo4j-driver.ts
│   ├── electron.ts
│   ├── websocket.ts
│   └── http.ts
├── helpers/           # Test helpers
│   ├── testUtils.ts
│   └── test-server.ts
├── setup.ts           # Global test setup
└── *.spec.js          # Playwright E2E tests
```

---

## Test Types

### Unit Tests

**Purpose:** Test individual components/modules in isolation.

**Location:** `/tests/unit/`

**Framework:** Vitest with jsdom

**Example:**

```typescript
import { describe, it, expect, beforeEach, vi } from 'vitest';
import { store } from '@/modules/Store.js';

describe('Store', () => {
  beforeEach(() => {
    vi.clearAllMocks();
  });

  it('should update state and notify listeners', async () => {
    const listener = vi.fn();
    store.subscribe(listener);

    store.setActiveModule('discuss');

    await nextMicrotask(); // Wait for async notification

    expect(listener).toHaveBeenCalled();
    expect(store.getState().activeModule).toBe('discuss');
  });
});
```

### Integration Tests

**Purpose:** Test interactions between components and real services.

**Location:** `/tests/integration/`

**Framework:** Vitest with Neo4j test container

**Example:**

```typescript
import { describe, it, expect, beforeAll, afterAll } from 'vitest';
import { createNeo4jDriver } from '@/lib/neo4j';

describe('Neo4j Integration', () => {
  let driver;

  beforeAll(async () => {
    driver = await createNeo4jDriver({
      uri: process.env.NEO4J_URI,
      username: process.env.NEO4J_USERNAME,
      password: process.env.NEO4J_PASSWORD,
    });
  });

  afterAll(async () => {
    await driver.close();
  });

  it('should create and retrieve a node', async () => {
    const session = driver.session();
    try {
      const result = await session.run(
        'CREATE (n:Test {name: $name}) RETURN n',
        { name: 'Test Node' },
      );
      expect(result.records).toHaveLength(1);
    } finally {
      await session.close();
    }
  });
});
```

### E2E Tests

**Purpose:** Test complete user workflows in Electron app.

**Location:** `/tests/`

**Framework:** Playwright

**Example:**

```javascript
const { test, expect } = require('@playwright/test');
const { _electron: electron } = require('playwright');

test('ODEI app loads successfully', async () => {
  const app = await electron.launch({ args: ['.'] });
  const window = await app.firstWindow();

  await expect(window).toHaveTitle(/ODEI/);

  await app.close();
});
```

---

## Running Tests

### Quick Commands

```bash
# Run all tests
npm test

# Unit tests only
npm run test:unit

# Unit tests in watch mode
npm run test:watch

# Integration tests
npm run test:integration

# E2E tests (Playwright)
npm run test:playwright

# All tests (unit + E2E)
npm run test:all

# Coverage report
npm run test:coverage

# Interactive UI
npm run test:ui
```

### Advanced Options

```bash
# Run specific test file
npx vitest run tests/unit/Store.spec.ts

# Run tests matching pattern
npx vitest run -t "Store"

# Run with coverage
npx vitest run --coverage

# Debug tests
npx vitest --inspect-brk

# Run Playwright in headed mode
npx playwright test --headed

# Run Playwright with UI
npx playwright test --ui
```

---

## Writing Tests

### Unit Test Template

```typescript
/**
 * Unit tests for [Component Name]
 * Target: 90%+ coverage
 */

import { describe, it, expect, beforeEach, afterEach, vi } from 'vitest';
import { ComponentUnderTest } from '@/modules/ComponentUnderTest';

describe('ComponentUnderTest', () => {
  let component: ComponentUnderTest;

  beforeEach(() => {
    // Setup: Create fresh instance
    component = new ComponentUnderTest();
  });

  afterEach(() => {
    // Cleanup: Clear mocks, timers, etc.
    vi.clearAllMocks();
  });

  describe('methodName', () => {
    it('should do expected behavior', () => {
      // Arrange
      const input = 'test';

      // Act
      const result = component.methodName(input);

      // Assert
      expect(result).toBe('expected');
    });

    it('should handle edge case', () => {
      // Test edge cases, errors, etc.
    });
  });
});
```

### Integration Test Template

```typescript
import { describe, it, expect, beforeAll, afterAll } from 'vitest';
import { createTestEnvironment } from '@helpers/test-server';

describe('Integration: [Feature Name]', () => {
  let env: ReturnType<typeof createTestEnvironment>;

  beforeAll(() => {
    env = createTestEnvironment();
    env.set('NEO4J_URI', 'bolt://localhost:7687');
  });

  afterAll(() => {
    env.reset();
  });

  it('should integrate correctly', async () => {
    // Test real service integration
  });
});
```

### E2E Test Template

```javascript
const { test, expect } = require('@playwright/test');
const { _electron: electron } = require('playwright');

test.describe('Feature Name', () => {
  let app;
  let window;

  test.beforeAll(async () => {
    app = await electron.launch({ args: ['.'] });
    window = await app.firstWindow();
  });

  test.afterAll(async () => {
    await app.close();
  });

  test('should perform user workflow', async () => {
    // Click, type, navigate, etc.
    await window.click('[data-testid="button"]');
    await expect(window.locator('[data-testid="result"]')).toBeVisible();
  });
});
```

---

## Test Utilities

### Mocks

Use pre-built mocks from `/tests/mocks/`:

```typescript
import { createMockNeo4jDriver, createMockNode } from '@mocks/neo4j-driver';
import { createMockIpcRenderer } from '@mocks/electron';
import { createMockWebSocket } from '@mocks/websocket';
import { createMockFetch } from '@mocks/http';

// Neo4j
const { driver, neo4jMock } = mockNeo4jDriver();

// Electron
const { ipcRenderer } = createMockElectron();

// WebSocket
const ws = createMockWebSocket('ws://localhost:8080');
ws.triggerOpen();
ws.triggerMessage({ type: 'update', data: {} });

// HTTP
const mockFetch = createMockFetch([
  createMockResponse({ status: 200, body: { success: true } }),
]);
```

### Fixtures

Use test data from `/tests/fixtures/`:

```typescript
import { foundationNodes, visionNodes, completeGraph } from '@fixtures/graph-data';
import { agentConfigs, conversationThreads, messages } from '@fixtures/agent-data';

// Use in tests
const valueNode = foundationNodes.value;
const discussAgent = agentConfigs.discuss;
```

### Helpers

Use test helpers from `/tests/helpers/`:

```typescript
import { waitFor, nextMicrotask, mockLocalStorage } from '@helpers/testUtils';
import { createTestMCPServer, waitForServer } from '@helpers/test-server';

// Wait for condition
await waitFor(() => element.isVisible(), 5000);

// Wait for microtask (Store notifications)
await nextMicrotask();

// Mock localStorage
const { store, mock } = mockLocalStorage();
localStorage.setItem('key', 'value');
expect(mock.setItem).toHaveBeenCalled();
```

---

## Coverage Requirements

### Thresholds

ODEI enforces minimum coverage thresholds:

- **Lines:** 70%
- **Functions:** 70%
- **Branches:** 65%
- **Statements:** 70%

### Viewing Coverage

```bash
# Generate HTML report
npm run test:coverage

# Open in browser
open test-results/coverage/index.html
```

### Coverage Tips

1. **Focus on critical paths** — Core business logic first
2. **Test error handling** — Cover edge cases and failures
3. **Mock external dependencies** — Isolate units effectively
4. **Avoid testing implementation details** — Test behavior, not code structure
5. **Use coverage as a guide** — Not a goal; quality > quantity

---

## CI/CD Integration

### GitHub Actions

Tests run automatically on:

- Push to `main` or `develop`
- Pull requests to `main` or `develop`

**Workflow:** `.github/workflows/test.yml`

**Jobs:**

1. **Unit Tests** — Vitest on Node 20.x & 22.x
2. **Integration Tests** — With Neo4j service
3. **E2E Tests** — Playwright on Ubuntu & macOS
4. **Type Check** — TypeScript compiler
5. **Lint** — ESLint + Prettier

### Artifacts

Test results and coverage reports are uploaded as artifacts:

- Unit test results (7 days)
- Integration test results (7 days)
- Playwright reports (7 days)
- Coverage reports (uploaded to Codecov)

### Status Badges

Add to README:

```markdown
![Tests](https://github.com/your-org/ODEI/workflows/Test%20Suite/badge.svg)
[![codecov](https://codecov.io/gh/your-org/ODEI/branch/main/graph/badge.svg)](https://codecov.io/gh/your-org/ODEI)
```

---

## Troubleshooting

### Common Issues

#### Tests timeout

```bash
# Increase timeout in vitest.config.ts
testTimeout: 30000  # 30 seconds
```

#### Neo4j connection fails in CI

```bash
# Wait for service to be ready
timeout 60 bash -c 'until curl -f http://localhost:7474; do sleep 2; done'
```

#### Playwright browser not found

```bash
# Install browsers
npx playwright install --with-deps
```

#### Module import errors

```bash
# Clear cache and reinstall
rm -rf node_modules package-lock.json
npm install
```

#### Coverage threshold not met

```bash
# Check which files need more coverage
npm run test:coverage -- --reporter=json --reporter=text

# Focus on files with low coverage
npx vitest run --coverage path/to/file.spec.ts
```

### Debugging Tests

#### Vitest

```bash
# Use Node inspector
npx vitest --inspect-brk

# Use VS Code debugger
# Add breakpoint, run "Debug Test" in VS Code
```

#### Playwright

```bash
# Use Playwright Inspector
PWDEBUG=1 npx playwright test

# Use headed mode
npx playwright test --headed

# Use UI mode
npx playwright test --ui
```

### Getting Help

1. **Check logs** — Test output contains detailed error messages
2. **Run tests locally** — Reproduce CI failures on your machine
3. **Isolate the test** — Run single test file to narrow down issue
4. **Check dependencies** — Ensure all services (Neo4j, etc.) are running
5. **Read error stack traces** — Follow the trail to root cause

---

## Best Practices

### General

- Write tests **before** fixing bugs (TDD for bug fixes)
- Keep tests **focused** — One behavior per test
- Use **descriptive names** — `it('should X when Y')` format
- **Arrange-Act-Assert** pattern for clarity
- **Clean up** after tests (close connections, clear timers)

### Unit Tests

- Mock **external dependencies** completely
- Test **public API** only, not internals
- Cover **happy path + error cases**
- Use **parameterized tests** for similar cases

### Integration Tests

- Use **real services** when possible (Neo4j, databases)
- **Isolate** tests (separate database/schema per test)
- **Clean up** data after tests
- Keep tests **fast** (optimize queries, limit data)

### E2E Tests

- Test **critical user workflows** only
- Use **data-testid** attributes for selectors
- Keep tests **independent** (no test order dependency)
- Handle **async operations** properly (waitFor, expect polling)

---

## Next Steps

- [View Example Tests](/tests/unit/Store.spec.ts)
- [Configure Coverage](/vitest.config.ts)
- [Check CI Status](/.github/workflows/test.yml)
- [Report Issues](https://github.com/your-org/ODEI/issues)

---

**Built with care for ODEI Symbiosis — Infrastructure Developer Mode**
