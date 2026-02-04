# ODEI Performance Guidelines

This document describes performance best practices, budgets, and anti-patterns for the ODEI Electron application.

## Table of Contents

- [Performance Budgets](#performance-budgets)
- [Memory Leak Prevention](#memory-leak-prevention)
- [Main Thread Work Limits](#main-thread-work-limits)
- [Debouncing and Throttling](#debouncing-and-throttling)
- [Frame Rate and Latency Budgets](#frame-rate-and-latency-budgets)
- [MCP Call Optimization](#mcp-call-optimization)
- [3D Graph Performance](#3d-graph-performance)
- [Event Listener Management](#event-listener-management)
- [Anti-Patterns](#anti-patterns)

---

## Performance Budgets

### Frame Rate (FPS)

| View                      | Target FPS    | Minimum Acceptable | Critical |
| ------------------------- | ------------- | ------------------ | -------- |
| Static UI                 | 60 FPS (16ms) | 30 FPS (33ms)      | <20 FPS  |
| 3D Graph (idle)           | 60 FPS        | 30 FPS             | <20 FPS  |
| 3D Graph (physics active) | 30 FPS (33ms) | 20 FPS (50ms)      | <15 FPS  |
| Terminal scrolling        | 60 FPS        | 30 FPS             | <20 FPS  |
| Animations                | 60 FPS        | 30 FPS             | <20 FPS  |

**Budget per frame**: 16.67ms (60 FPS) or 33.33ms (30 FPS)

### Memory Usage

| Metric          | Target | Acceptable | Critical |
| --------------- | ------ | ---------- | -------- |
| JavaScript Heap | <150MB | <300MB     | >500MB   |
| Total Memory    | <300MB | <600MB     | >1GB     |
| Heap growth/min | <5MB   | <10MB      | >20MB    |
| Event listeners | <100   | <200       | >500     |

**Check memory in DevTools**: `performance.memory.usedJSHeapSize / 1024 / 1024`

### Latency

| Operation                  | Fast   | Acceptable | Slow    | Critical |
| -------------------------- | ------ | ---------- | ------- | -------- |
| UI interaction (click)     | <50ms  | <100ms     | <200ms  | >500ms   |
| Agent tab switch           | <100ms | <200ms     | <500ms  | >1000ms  |
| Terminal output            | <16ms  | <50ms      | <100ms  | >200ms   |
| MCP call                   | <200ms | <1000ms    | <3000ms | >10000ms |
| Graph render (first paint) | <500ms | <1000ms    | <2000ms | >5000ms  |

---

## Memory Leak Prevention

Memory leaks are the #1 cause of ODEI freezes. Follow these patterns rigorously.

### Event Listener Cleanup

**Problem**: Every `addEventListener` creates a closure that keeps references alive.

**Pattern: Store cleanup functions**

```javascript
class MyComponent {
  constructor() {
    this.disposables = [];
  }

  init() {
    const clickHandler = () => this.handleClick();
    document.addEventListener('click', clickHandler);

    // CRITICAL: Store cleanup function
    this.disposables.push(() => {
      document.removeEventListener('click', clickHandler);
    });
  }

  destroy() {
    // Clean up all listeners
    this.disposables.forEach((fn) => fn());
    this.disposables = [];
  }
}
```

**Example: UIManager conversation row listeners**

```javascript
// BEFORE (LEAKED):
updateRecentConversationsUI() {
  rows.forEach(row => {
    row.addEventListener('click', handler); // Never removed!
  });
}

// AFTER (FIXED):
updateRecentConversationsUI() {
  // Clean up old listeners first
  this.conversationRowDisposables.forEach(fn => fn());
  this.conversationRowDisposables = [];

  // Add new listeners and store cleanup
  rows.forEach(row => {
    const handler = () => loadConversation(row.id);
    row.addEventListener('click', handler);
    this.conversationRowDisposables.push(
      () => row.removeEventListener('click', handler)
    );
  });
}
```

### DOM Node References

**Problem**: Keeping references to removed DOM nodes prevents GC.

**Pattern: Clear references on cleanup**

```javascript
class Component {
  constructor() {
    this.element = document.createElement('div');
    this.childElements = [];
  }

  render() {
    this.element.innerHTML = '';
    this.childElements = []; // Clear old references

    // Create new elements
    const child = document.createElement('span');
    this.element.appendChild(child);
    this.childElements.push(child);
  }

  destroy() {
    // Clear references before removing from DOM
    this.childElements = [];
    this.element.remove();
    this.element = null;
  }
}
```

### Interval and Timer Cleanup

**Problem**: Timers keep running after component destroyed.

**Pattern: Clear in destroy()**

```javascript
class Component {
  init() {
    this.interval = setInterval(() => this.update(), 1000);
    this.timeout = setTimeout(() => this.load(), 500);
  }

  destroy() {
    if (this.interval) {
      clearInterval(this.interval);
      this.interval = null;
    }

    if (this.timeout) {
      clearTimeout(this.timeout);
      this.timeout = null;
    }
  }
}
```

### requestAnimationFrame Cleanup

**Problem**: RAF loops continue after view hidden, wasting CPU/battery.

**Pattern: Cancel RAF on pause**

```javascript
class Renderer {
  constructor() {
    this.rafId = null;
    this._isPaused = false;
  }

  start() {
    const loop = () => {
      // CRITICAL: Check pause state
      if (this._isPaused) {
        this.rafId = null;
        return; // Stop loop
      }

      this.render();

      // Only reschedule if still active
      if (!this._isPaused) {
        this.rafId = requestAnimationFrame(loop);
      }
    };

    this.rafId = requestAnimationFrame(loop);
  }

  pause() {
    this._isPaused = true;
    if (this.rafId) {
      cancelAnimationFrame(this.rafId);
      this.rafId = null;
    }
  }

  resume() {
    if (!this._isPaused) return;
    this._isPaused = false;
    this.start(); // Restart loop
  }

  destroy() {
    this.pause();
  }
}
```

**Example: ThreeGraphRenderer pause/resume**

```javascript
// Pause when switching away from Memory view
switchAgent(agent) {
  if (prevAgent === 'memory') {
    memoryController.graphRenderer.pauseAnimation();
  }

  if (agent === 'memory') {
    memoryController.graphRenderer.resumeAnimation();
  }
}
```

### WebGL Resource Disposal

**Problem**: WebGL textures, geometries, materials aren't auto-collected.

**Pattern: Explicit disposal**

```javascript
class ThreeGraphRenderer {
  createNode(node) {
    const geometry = new THREE.SphereGeometry(radius, 16, 16);
    const material = new THREE.MeshLambertMaterial({ color });
    const mesh = new THREE.Mesh(geometry, material);

    // Track for disposal
    this._customNodeObjects.set(node.id, {
      mesh,
      geometry,
      material,
    });

    return mesh;
  }

  dispose() {
    // CRITICAL: Dispose WebGL resources
    this._customNodeObjects.forEach((obj) => {
      if (obj.geometry) obj.geometry.dispose();

      if (obj.material) {
        if (Array.isArray(obj.material)) {
          obj.material.forEach((m) => m.dispose());
        } else {
          obj.material.dispose();
        }
      }
    });

    this._customNodeObjects.clear();
  }
}
```

### Cache Size Limits

**Problem**: Unbounded caches grow until out-of-memory.

**Pattern: LRU cache with max size**

```javascript
const CACHE_MAX_SIZE = 500;
const cache = new Map();
const lru = []; // Access order

function cacheGet(key) {
  const value = cache.get(key);
  if (value) {
    // Move to end (most recently used)
    const idx = lru.indexOf(key);
    if (idx > -1) lru.splice(idx, 1);
    lru.push(key);
  }
  return value;
}

function cacheSet(key, value) {
  // Evict oldest if at capacity
  while (cache.size >= CACHE_MAX_SIZE && lru.length > 0) {
    const evictKey = lru.shift();
    const evictValue = cache.get(evictKey);

    // Dispose if has cleanup method
    if (evictValue?.dispose) evictValue.dispose();

    cache.delete(evictKey);
  }

  cache.set(key, value);
  lru.push(key);
}
```

**Example: ThreeGraphRenderer label cache**

See `labelCacheGet()` and `labelCacheSet()` in `/src/modules/ThreeGraphRenderer.js`.

---

## Main Thread Work Limits

The main thread must stay responsive for UI. Budget main thread work carefully.

### Per-Frame Budget

- **Target**: <10ms per frame (leaves 6ms for browser/Electron overhead)
- **Maximum**: <16ms per frame (60 FPS floor)
- **Critical**: >33ms per frame (<30 FPS, users notice lag)

### Expensive Operations

| Operation                      | Approx Cost              | Mitigation                                  |
| ------------------------------ | ------------------------ | ------------------------------------------- |
| DOM query (`querySelectorAll`) | 0.1-1ms per 100 elements | Cache results, use IDs                      |
| DOM insertion                  | 0.5-5ms per 100 nodes    | Use DocumentFragment                        |
| Style recalc                   | 1-10ms                   | Batch style changes, avoid layout thrashing |
| Layout (reflow)                | 5-50ms                   | Read then write, avoid forced sync layout   |
| JSON.parse (1MB)               | 10-30ms                  | Stream or chunk large JSON                  |
| Array.sort (10k items)         | 1-5ms                    | Use Web Worker for large sorts              |
| String.replace (regex, 1MB)    | 10-50ms                  | Chunk or use simpler patterns               |

### Forced Synchronous Layout (Reflow)

**Anti-pattern: Read-write-read-write (thrashing)**

```javascript
// BAD: Forces 3 layouts
for (const el of elements) {
  const height = el.offsetHeight; // READ (forces layout)
  el.style.height = height + 10 + 'px'; // WRITE
  // Next iteration READs again, forcing another layout
}
```

**Pattern: Batch reads, then batch writes**

```javascript
// GOOD: 1 layout
const heights = elements.map((el) => el.offsetHeight); // READ all
elements.forEach((el, i) => {
  el.style.height = heights[i] + 10 + 'px'; // WRITE all
});
```

### Offload to Web Workers

Move CPU-intensive work off main thread:

```javascript
// worker.js
self.onmessage = (e) => {
  const sorted = e.data.sort((a, b) => a - b);
  self.postMessage(sorted);
};

// main.js
const worker = new Worker('worker.js');
worker.postMessage(largeArray);
worker.onmessage = (e) => {
  const sorted = e.data;
  // Update UI
};
```

**Limitations**:

- Can't access DOM from worker
- Can't use Electron IPC from worker
- Must serialize data (no functions, DOM nodes)

---

## Debouncing and Throttling

Use debouncing/throttling for high-frequency events.

### Debounce

**Use when**: You want to delay execution until events stop firing.

**Examples**: Search input, window resize

```javascript
import { debounce } from './utils/performance.js';

const handleInput = debounce((value) => {
  searchDatabase(value);
}, 300); // Wait 300ms after typing stops

input.addEventListener('input', (e) => handleInput(e.target.value));
```

**Implementation**:

```javascript
export function debounce(fn, delayMs) {
  let timer;
  return function (...args) {
    clearTimeout(timer);
    timer = setTimeout(() => fn.apply(this, args), delayMs);
  };
}
```

### Throttle

**Use when**: You want to limit execution rate, but still fire during events.

**Examples**: Scroll handler, drag handler

```javascript
import { throttle } from './utils/performance.js';

const handleScroll = throttle(() => {
  updateScrollIndicator();
}, 100); // Max 10 times per second

window.addEventListener('scroll', handleScroll);
```

**Implementation**:

```javascript
export function throttle(fn, delayMs) {
  let lastCall = 0;
  return function (...args) {
    const now = Date.now();
    if (now - lastCall < delayMs) return;
    lastCall = now;
    fn.apply(this, args);
  };
}
```

### RAF Debounce

**Use when**: Synchronize with browser paint cycle.

**Examples**: Layout updates, canvas rendering

```javascript
import { rafDebounce } from './utils/performance.js';

const handleResize = rafDebounce(() => {
  updateLayout();
});

window.addEventListener('resize', handleResize);
```

**Implementation**:

```javascript
export function rafDebounce(fn) {
  let rafId = null;
  return function (...args) {
    if (rafId) cancelAnimationFrame(rafId);
    rafId = requestAnimationFrame(() => fn.apply(this, args));
  };
}
```

### When to Use Which

| Pattern      | Latency                | Accuracy                              | Use Case                   |
| ------------ | ---------------------- | ------------------------------------- | -------------------------- |
| Debounce     | High (waits for quiet) | High (always fires after events stop) | Search, form validation    |
| Throttle     | Low (fires during)     | Medium (may skip events)              | Scroll, drag, mouse move   |
| RAF Debounce | Medium (next frame)    | High (synced with paint)              | Layout, canvas, animations |

---

## Frame Rate and Latency Budgets

### 60 FPS Target

**Budget**: 16.67ms per frame

**Breakdown**:

- Browser overhead: ~4ms (compositor, paint, etc.)
- **Your budget**: 12ms for JavaScript execution
- Leave 0.67ms buffer for jank prevention

**If you exceed**: Frame drops to 30 FPS (33.33ms)

### 30 FPS Acceptable (3D Graph Active)

During ForceGraph3D physics simulation, 30 FPS is acceptable.

**Budget**: 33.33ms per frame

**Configuration** (in ThreeGraphRenderer):

```javascript
// Aggressive physics settings for faster convergence
this.Graph3D.d3AlphaDecay(0.05) // Faster cooldown (default: 0.0228)
  .d3VelocityDecay(0.6) // Nodes slow down faster (default: 0.4)
  .warmupTicks(10) // Reduced pre-computation (default: 0)
  .cooldownTicks(15) // Faster stabilization (default: 0)
  .cooldownTime(1000); // 1 second max (default: 15000ms)
```

### Latency Budgets by Interaction

| Interaction | Target       | User Perception                             |
| ----------- | ------------ | ------------------------------------------- |
| <100ms      | Instant      | Feels direct, no delay                      |
| 100-300ms   | Slight delay | Acceptable, user notices very brief lag     |
| 300-1000ms  | Noticeable   | Feels sluggish, user questions if it worked |
| >1000ms     | Slow         | User may click again or think app is frozen |

**Examples**:

- Button click → visual feedback: <50ms
- Agent tab switch → view rendered: <200ms
- Search input → results shown: <300ms (debounced)
- MCP call → data displayed: <1000ms

---

## MCP Call Optimization

MCP calls are the most common source of UI lag. Optimize carefully.

### Concurrency Limit

**Budget**: Max 6 concurrent MCP requests

**Rationale**:

- Neo4j connection pool: 10 connections
- Leave 4 for other MCP servers
- Prevents overwhelming server

**Configuration** (in mcpClient):

```javascript
this.maxConcurrent = 6;
```

### Timeout

**Budget**: 10 seconds per call

**Rationale**:

- Most queries return in <1 second
- Complex aggregations may take 3-5 seconds
- 10 seconds catches true hangs without failing fast queries

**Configuration**:

```javascript
this.toolTimeoutMs = 10_000; // 10 seconds
```

### Retry Strategy

**Budget**: 1 automatic retry with 400ms backoff

**Rationale**:

- Transient network errors common
- 1 retry is enough (2 total attempts)
- 400ms backoff gives network time to recover

**Configuration**:

```javascript
this.maxRetries = 1;
this.backoffMs = 400;
```

### Parallel Calls

**Anti-pattern: Uncontrolled parallelism**

```javascript
// BAD: Spawns 50 concurrent requests!
const promises = agents.map((agent) => mcpClient.callTool('odei-neo4j', 'get-status', { agent }));
await Promise.all(promises);
```

**Pattern: Semaphore automatically limits**

```javascript
// GOOD: mcpClient limits to 6 concurrent, queues rest
const promises = agents.map((agent) => mcpClient.callTool('odei-neo4j', 'get-status', { agent }));
await Promise.all(promises);
// First 6 execute immediately, rest queue up
```

### Caching (Planned)

Cache expensive MCP results for 30-60 seconds:

```javascript
const cache = new Map();

async function cachedMCPCall(server, tool, params) {
  const key = `${server}:${tool}:${JSON.stringify(params)}`;
  const cached = cache.get(key);

  if (cached && Date.now() - cached.timestamp < 30_000) {
    return cached.result; // Return cached result
  }

  const result = await mcpClient.callTool(server, tool, params);

  cache.set(key, { result, timestamp: Date.now() });
  return result;
}
```

---

## 3D Graph Performance

ForceGraph3D is the most performance-critical component.

### Physics Simulation

**Problem**: Physics runs on main thread, blocks UI.

**Solution: Aggressive stabilization settings**

```javascript
// CRITICAL: Stop physics after 1 second
this.Graph3D.d3AlphaDecay(0.05) // Faster convergence
  .d3VelocityDecay(0.6) // Nodes slow down faster
  .warmupTicks(10) // Minimal pre-computation
  .cooldownTicks(15) // Fast stabilization
  .cooldownTime(1000) // 1 second max
  .onEngineStop(() => {
    this._physicsStable = true; // Flag for optimization
  });
```

**Budget**: 1 second max for physics to stabilize.

### RAF Loop Optimization

**Problem**: RAF loop runs 60x/second even when paused.

**Solution: Stop RAF when hidden**

```javascript
// CRITICAL: Check pause state before any work
if (this._isPaused || this._disposed || this._contextLost) {
  this._labelRaf = null;
  return; // Stop loop
}

// PERFORMANCE: Also check if container is visible
if (this.container && this.container.offsetParent === null) {
  this._labelRaf = null;
  return; // Container is hidden (display:none)
}

// Do actual work
this._updateLabelPositions();

// Only reschedule if still active
if (!this._isPaused && !this._disposed && !this._contextLost) {
  this._labelRaf = requestAnimationFrame(this._rafLabelUpdater);
}
```

### Pause/Resume on Tab Switch

**Critical**: Pause graph when switching to another view.

```javascript
// UIManager.switchAgent()
if (previousModule === 'memory' && memoryController) {
  memoryController.deactivate(); // Pauses graph
}

if (targetAgent === 'memory') {
  memoryController.activate(); // Resumes graph
}
```

### Node Count Limits

| Node Count | Performance          | Recommendation     |
| ---------- | -------------------- | ------------------ |
| <100       | Excellent (60 FPS)   | No issues          |
| 100-500    | Good (30-60 FPS)     | Acceptable         |
| 500-1000   | Degraded (20-30 FPS) | Paginate or filter |
| >1000      | Poor (<20 FPS)       | **Must paginate**  |

### Link Rendering

**Budget**: <2000 links for good performance

**Pattern: Hide links when too many**

```javascript
if (links.length > 2000) {
  this.Graph3D.linkOpacity(0.1); // Nearly invisible
  this.Graph3D.linkDirectionalParticles(0); // No particles
}
```

### Label Optimization

**Problem**: Canvas label textures created 60x/second.

**Solution: LRU cache with 500 entry limit**

```javascript
// See ThreeGraphRenderer.js labelCacheGet/Set
const LABEL_CACHE_MAX_SIZE = 500;

// Evict oldest when at capacity
while (cache.size >= LABEL_CACHE_MAX_SIZE) {
  const evictKey = lru.shift();
  cache.get(evictKey)?.dispose(); // Dispose WebGL texture
  cache.delete(evictKey);
}
```

### Geometry/Material Disposal

**Critical**: Dispose on every graph data update.

```javascript
// Clear old node objects before creating new ones
this._customNodeObjects.forEach(obj => {
  if (obj.geometry) obj.geometry.dispose();
  if (obj.material) obj.material.dispose();
});
this._customNodeObjects.clear();

// Create new nodes
this.Graph3D.nodeThreeObject(node => {
  const geometry = new THREE.SphereGeometry(...);
  const material = new THREE.MeshLambertMaterial(...);
  const mesh = new THREE.Mesh(geometry, material);

  // Track for next disposal
  this._customNodeObjects.set(node.id, { geometry, material, mesh });

  return mesh;
});
```

---

## Event Listener Management

Listeners are the #1 cause of memory leaks. Manage lifecycle rigorously.

### Pattern: Disposables Array

**Every component that adds listeners must have:**

```javascript
class Component {
  constructor() {
    this.disposables = [];
  }

  init() {
    const handler = () => this.handleEvent();
    document.addEventListener('event', handler);

    this.disposables.push(() => {
      document.removeEventListener('event', handler);
    });
  }

  destroy() {
    this.disposables.forEach((fn) => {
      try {
        fn();
      } catch (err) {
        console.error('Cleanup error:', err);
      }
    });
    this.disposables = [];
  }
}
```

### Pattern: Subscription Return Value

**Many APIs return unsubscribe functions:**

```javascript
const unsub1 = store.subscribe(listener);
const unsub2 = mcpClient.onMetrics(listener);

this.disposables.push(unsub1, unsub2);
```

### Pattern: Separate Array for Dynamic Listeners

**For listeners that change frequently (e.g., conversation rows):**

```javascript
class UIManager {
  constructor() {
    this.disposables = []; // Global listeners (live forever)
    this.conversationRowDisposables = []; // Dynamic listeners (recreated on render)
  }

  updateConversations() {
    // CRITICAL: Clean up old listeners first
    this.conversationRowDisposables.forEach((fn) => fn());
    this.conversationRowDisposables = [];

    // Add new listeners
    rows.forEach((row) => {
      const handler = () => loadConversation(row);
      row.addEventListener('click', handler);
      this.conversationRowDisposables.push(() => row.removeEventListener('click', handler));
    });
  }

  destroy() {
    // Clean both arrays
    this.disposables.forEach((fn) => fn());
    this.conversationRowDisposables.forEach((fn) => fn());
    this.disposables = [];
    this.conversationRowDisposables = [];
  }
}
```

### Listener Count Budget

| Component                   | Target | Max  | Critical |
| --------------------------- | ------ | ---- | -------- |
| UIManager (global)          | <50    | <100 | >200     |
| Conversation rows (dynamic) | <30    | <50  | >100     |
| Graph renderer              | <10    | <20  | >50      |
| **Total app**               | <100   | <200 | >500     |

**Check in DevTools:**

```javascript
// No direct API, but can estimate
document.querySelectorAll('*').length; // All DOM nodes
// If growing unbounded, listeners probably leaking
```

---

## Anti-Patterns

### ❌ Direct State Mutation

```javascript
// BAD
store.state.activeModule = 'memory';

// GOOD
store.setActiveModule('memory');
```

### ❌ Forgetting to Unsubscribe

```javascript
// BAD
store.subscribe((state) => updateUI(state));
// Listener lives forever, even after component destroyed

// GOOD
const unsub = store.subscribe((state) => updateUI(state));
this.disposables.push(unsub);
```

### ❌ Synchronous Layout Thrashing

```javascript
// BAD
elements.forEach((el) => {
  const height = el.offsetHeight; // Forces layout
  el.style.height = height + 10 + 'px'; // Invalidates layout
}); // Next iteration forces layout again = thrashing

// GOOD
const heights = elements.map((el) => el.offsetHeight); // Read all
elements.forEach((el, i) => {
  el.style.height = heights[i] + 10 + 'px'; // Write all
});
```

### ❌ Unbounded Caches

```javascript
// BAD
const cache = new Map();
function cacheResult(key, value) {
  cache.set(key, value); // Grows forever!
}

// GOOD
const CACHE_MAX = 500;
const cache = new Map();
const lru = [];

function cacheResult(key, value) {
  while (cache.size >= CACHE_MAX) {
    const evict = lru.shift();
    cache.delete(evict);
  }
  cache.set(key, value);
  lru.push(key);
}
```

### ❌ RAF Without Pause Check

```javascript
// BAD
function loop() {
  render();
  requestAnimationFrame(loop); // Runs forever, even when hidden!
}
loop();

// GOOD
function loop() {
  if (this._isPaused) return; // Stop when paused

  render();

  if (!this._isPaused) {
    requestAnimationFrame(loop); // Only reschedule if active
  }
}
```

### ❌ Ignoring Disposal

```javascript
// BAD
class Component {
  init() {
    const geometry = new THREE.SphereGeometry(10, 16, 16);
    const material = new THREE.MeshBasicMaterial({ color: 0xff0000 });
    this.mesh = new THREE.Mesh(geometry, material);
    scene.add(this.mesh);
  }
  // No cleanup - memory leak!
}

// GOOD
class Component {
  init() {
    this.geometry = new THREE.SphereGeometry(10, 16, 16);
    this.material = new THREE.MeshBasicMaterial({ color: 0xff0000 });
    this.mesh = new THREE.Mesh(this.geometry, this.material);
    scene.add(this.mesh);
  }

  destroy() {
    scene.remove(this.mesh);
    this.geometry.dispose();
    this.material.dispose();
    this.mesh = null;
    this.geometry = null;
    this.material = null;
  }
}
```

---

## Debugging Performance Issues

### Chrome DevTools Performance Tab

1. Open DevTools → Performance tab
2. Click Record
3. Interact with app (reproduce lag)
4. Stop recording
5. Look for:
   - Long tasks (>50ms yellow blocks)
   - Forced reflows (purple "Recalculate Style" or "Layout")
   - Memory sawtooth (GC thrashing)

### Memory Profiler

1. DevTools → Memory tab
2. Take heap snapshot
3. Interact with app
4. Take another snapshot
5. Compare snapshots:
   - Look for retained objects (should be freed)
   - Check "Detached DOM tree" (memory leak)
   - Look for growing arrays, maps, sets

### FreezeDetector

```javascript
// Check recent freezes
window.freezeDetector.getLog();

// Track operation
const end = window.freezeDetector.trackOperation('my-op', { meta: 'data' });
// ... do expensive work ...
end();
```

### CriticalDiagnostic Overlay

Press `` (backtick) to toggle diagnostic overlay:

- FPS counter
- Memory usage
- Last frame timestamp
- WebGL status
- Event log (freezes highlighted in red)

---

## Summary Checklist

Before shipping performance-critical code:

- [ ] All event listeners stored in `disposables` array
- [ ] `destroy()` method cleans up all listeners
- [ ] RAF loops check `_isPaused` flag before work
- [ ] RAF loops cancelled when component destroyed
- [ ] WebGL resources (geometry, material, texture) disposed
- [ ] Caches have max size limits (LRU eviction)
- [ ] High-frequency events (resize, scroll, input) debounced/throttled
- [ ] MCP calls wrapped in try-catch with error handling
- [ ] 3D graph paused when view hidden
- [ ] No layout thrashing (batch reads, then writes)
- [ ] FreezeDetector integrated for operation tracking
- [ ] Performance budgets met (FPS, latency, memory)

---

## Related Documents

- [ADR-002: Freeze Detection](./adr/002-freeze-detection.md)
- [ADR-003: MCP Server Architecture](./adr/003-mcp-server-architecture.md)
- [Store API](./api/Store.md)
- [mcpClient API](./api/mcpClient.md)
- [UIManager API](./api/UIManager.md)
