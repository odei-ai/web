# ODEI Electron IPC API Reference

Complete reference for all IPC APIs exposed via `window.odei.*` and `window.electronAPI.*`

## Table of Contents

- [Primary API: window.odei.\*](#primary-api-windowodei)
  - [Terminal Operations](#terminal-operations)
  - [MCP Data Access](#mcp-data-access)
  - [IPC Event Bus](#ipc-event-bus)
  - [Menu Events](#menu-events)
  - [Observability Events](#observability-events)
  - [Health Monitoring](#health-monitoring)
  - [External Links](#external-links)
  - [Clipboard Operations](#clipboard-operations)
  - [Image Save](#image-save)
  - [Token Counting](#token-counting)
  - [Companion Terminal](#companion-terminal)
  - [Lifecycle](#lifecycle)
- [Legacy API: window.electronAPI.\*](#legacy-api-windowelectronapi-deprecated)
- [Complete Workflows](#complete-workflows)
- [Security Notes](#security-notes)

---

## Primary API: window.odei.\*

The primary API surface for all renderer-to-main IPC communication. All operations are exposed via the global `window.odei` object in the renderer process.

### Terminal Operations

Control and interact with agent terminal processes.

#### `window.odei.terminal.create(moduleId, cols, rows)`

Creates/resizes a terminal for the specified agent module.

**Parameters:**

- `moduleId` (string) - Agent identifier ('discuss', 'decisions', 'execute', 'mind')
- `cols` (number) - Terminal columns
- `rows` (number) - Terminal rows

**Returns:** void (fire-and-forget)

**Example:**

```javascript
window.odei.terminal.create('discuss', 80, 30);
```

---

#### `window.odei.terminal.write(moduleId, data)`

Write input data to an agent's terminal stdin.

**Parameters:**

- `moduleId` (string) - Agent identifier
- `data` (string) - Input data to send

**Returns:** void

**Example:**

```javascript
window.odei.terminal.write('discuss', 'Hello, agent!\n');
```

---

#### `window.odei.terminal.resize(moduleId, cols, rows)`

Resize an agent's terminal.

**Parameters:**

- `moduleId` (string) - Agent identifier
- `cols` (number) - New column count
- `rows` (number) - New row count

**Returns:** void

**Example:**

```javascript
window.odei.terminal.resize('discuss', 120, 40);
```

---

#### `window.odei.terminal.kill(moduleId)`

Kill an agent's terminal process.

**Parameters:**

- `moduleId` (string) - Agent identifier

**Returns:** void

**Example:**

```javascript
window.odei.terminal.kill('discuss');
```

---

#### `window.odei.terminal.restart(moduleId)`

Restart an agent's terminal process (kill + start with 500ms delay).

**Parameters:**

- `moduleId` (string) - Agent identifier

**Returns:** void

**Example:**

```javascript
window.odei.terminal.restart('discuss');
```

---

#### `window.odei.terminal.onData(moduleId, callback)`

Subscribe to terminal output data for a specific agent.

**Parameters:**

- `moduleId` (string) - Agent identifier to filter
- `callback` (function) - `(data: string) => void` - Called when output arrives

**Returns:** `() => void` - Cleanup function to remove listener

**Example:**

```javascript
const cleanup = window.odei.terminal.onData('discuss', (data) => {
  console.log('Received from discuss:', data);
});

// Later, when unmounting:
cleanup();
```

---

### MCP Data Access

Call MCP (Model Context Protocol) tools from the renderer process.

#### `window.odei.data.get(serverName, toolName, params)`

Call an MCP tool on a registered server.

**Parameters:**

- `serverName` (string) - MCP server name (e.g., 'odei-neo4j', 'odei-history')
- `toolName` (string) - Tool identifier (e.g., 'odei.neo4j.memory.recall.v1')
- `params` (object) - Tool-specific parameters

**Returns:** `Promise<any>` - Tool result

**Example:**

```javascript
const memories = await window.odei.data.get('odei-neo4j', 'odei.neo4j.memory.recall.v1', {
  query: 'Recent conversations',
  limit: 10,
});
console.log('Memories:', memories);
```

---

### IPC Event Bus

Subscribe to custom IPC events from the main process.

#### `window.odei.ipc.on(event, callback)`

Subscribe to an IPC event.

**Parameters:**

- `event` (string) - Event name (e.g., 'menu:copy', 'menu:paste')
- `callback` (function) - `(data: any) => void` - Event handler

**Returns:** `() => void` - Cleanup function to remove listener

**Example:**

```javascript
const cleanup = window.odei.ipc.on('custom:event', (data) => {
  console.log('Custom event received:', data);
});

// Later:
cleanup();
```

---

### Menu Events

Handle menu-triggered events (previously undocumented).

#### `window.odei.ipc.on('menu:copy', callback)`

Triggered when user selects Edit → Copy or presses Cmd+C/Ctrl+C.

**Important:** The main process intercepts keyboard shortcuts BEFORE the system handles them via `before-input-event`. This ensures custom copy/paste logic runs instead of default browser behavior.

**Example:**

```javascript
window.odei.ipc.on('menu:copy', async () => {
  const selectedText = getSelectedText(); // Your implementation
  await window.odei.clipboard.writeText(selectedText);
  console.log('Copied to clipboard');
});
```

---

#### `window.odei.ipc.on('menu:paste', callback)`

Triggered when user selects Edit → Paste or presses Cmd+V/Ctrl+V.

**Example:**

```javascript
window.odei.ipc.on('menu:paste', async () => {
  const formats = await window.odei.clipboard.availableFormats();

  if (formats.hasImage) {
    const imageData = await window.odei.clipboard.readImage();
    handleImagePaste(imageData); // Your implementation
  } else if (formats.hasText) {
    const text = await window.odei.clipboard.readText();
    handleTextPaste(text); // Your implementation
  }
});
```

---

### Observability Events

Monitor agent activity, health, and context usage.

#### `window.odei.onAgentOutput(callback)`

Subscribe to all agent terminal output events.

**Parameters:**

- `callback` (function) - `({agent: string, data: string}) => void`

**Returns:** `() => void` - Cleanup function

**Example:**

```javascript
const cleanup = window.odei.onAgentOutput(({ agent, data }) => {
  console.log(`[${agent}]:`, data);
});
```

---

#### `window.odei.onAgentError(callback)`

Subscribe to agent error events.

**Parameters:**

- `callback` (function) - `({agent: string, error: string}) => void`

**Returns:** `() => void` - Cleanup function

**Example:**

```javascript
const cleanup = window.odei.onAgentError(({ agent, error }) => {
  console.error(`[${agent}] Error:`, error);
});
```

---

#### `window.odei.onAgentEvent(callback)`

Subscribe to agent control frame events (handoffs, state changes, etc.).

**Parameters:**

- `callback` (function) - `({agent: string, type: string, ...event}) => void`

**Returns:** `() => void` - Cleanup function

**Example:**

```javascript
const cleanup = window.odei.onAgentEvent((event) => {
  if (event.type === 'HANDOFF') {
    console.log(`Handoff from ${event.from} to ${event.to}`);
  }
});
```

---

#### `window.odei.onHealthUpdate(callback)`

Subscribe to MCP server health updates.

**Parameters:**

- `callback` (function) - `({status: string, uptime: number, memory: object, servers: object}) => void`

**Returns:** `() => void` - Cleanup function

**Example:**

```javascript
const cleanup = window.odei.onHealthUpdate((health) => {
  console.log('Health:', health.status);
  console.log('Servers:', health.servers);
});
```

---

#### `window.odei.onContextUsage(callback)`

Subscribe to agent context window usage events.

**Parameters:**

- `callback` (function) - `({agent: string, tokens: number}) => void`

**Returns:** `() => void` - Cleanup function

**Example:**

```javascript
const cleanup = window.odei.onContextUsage(({ agent, tokens }) => {
  console.log(`${agent} is using ${tokens} tokens`);
});
```

---

#### `window.odei.onContextAdd(callback)`

Subscribe to agent context additions (when agents call MCP tools).

**Parameters:**

- `callback` (function) - `({agent, serverName, toolName, params, result, timestamp}) => void`

**Returns:** `() => void` - Cleanup function

**Example:**

```javascript
const cleanup = window.odei.onContextAdd((item) => {
  console.log(`${item.agent} called ${item.toolName}`);
});
```

---

### Health Monitoring

Query current health status of all MCP servers.

#### `window.odei.health.get()`

Get current health snapshot.

**Returns:** `Promise<{status: string, uptime: number, memory: object, servers: object}>`

**Example:**

```javascript
const health = await window.odei.health.get();
console.log('Status:', health.status);
console.log('Servers:', health.servers);
```

**Response Format:**

```javascript
{
  status: 'healthy',
  uptime: 123456,
  memory: { heapUsed: 50000000, heapTotal: 100000000 },
  servers: {
    'odei-neo4j': { status: 'healthy', lastCheck: 1234567890 },
    'odei-history': { status: 'healthy', lastCheck: 1234567890 }
  }
}
```

---

### External Links

Open URLs in the default browser.

#### `window.odei.links.openExternal(url)`

Open a URL in the system's default browser.

**Security:** URLs are validated against an allowlist. Non-whitelisted domains trigger a confirmation dialog.

**Allowed domains:**

- github.com
- anthropic.com
- claude.com
- modelcontextprotocol.io
- electronjs.org
- neo4j.com
- xtermjs.org

**Parameters:**

- `url` (string) - URL to open

**Returns:** void

**Example:**

```javascript
window.odei.links.openExternal('https://github.com/anthropics');
```

---

### Clipboard Operations

Read and write clipboard data (text and images).

#### `window.odei.clipboard.writeText(text)`

Write text to the system clipboard.

**Parameters:**

- `text` (string) - Text to copy (max 100MB)

**Returns:** `Promise<{success: boolean, size?: number, error?: string}>`

**Example:**

```javascript
const result = await window.odei.clipboard.writeText('Hello, clipboard!');
if (result.success) {
  console.log(`Copied ${result.size} bytes`);
}
```

---

#### `window.odei.clipboard.readText()`

Read text from the system clipboard.

**Returns:** `Promise<string>` - Clipboard text content

**Example:**

```javascript
const text = await window.odei.clipboard.readText();
console.log('Clipboard text:', text);
```

---

#### `window.odei.clipboard.readImage()`

Read image from the system clipboard.

**Returns:** `Promise<{dataUrl: string, width: number, height: number, sizeKB: number} | null>`

**Example:**

```javascript
const imageData = await window.odei.clipboard.readImage();
if (imageData) {
  console.log(`Image: ${imageData.width}x${imageData.height}, ${imageData.sizeKB}KB`);
  // imageData.dataUrl is a base64 data URL: "data:image/png;base64,..."
}
```

---

#### `window.odei.clipboard.availableFormats()`

Check what formats are available in the clipboard.

**Returns:** `Promise<{formats: string[], hasText: boolean, hasImage: boolean}>`

**Example:**

```javascript
const formats = await window.odei.clipboard.availableFormats();
console.log('Has text:', formats.hasText);
console.log('Has image:', formats.hasImage);
console.log('All formats:', formats.formats);
```

---

### Image Save

Save base64-encoded images to temporary storage.

#### `window.odei.saveImage(base64Data, ext)`

Save a base64-encoded image to `/tmp/odei-images/`.

**Parameters:**

- `base64Data` (string) - Base64-encoded image data (WITHOUT data URL prefix)
- `ext` (string) - File extension (e.g., 'png', 'jpg')

**Returns:** `Promise<{path: string}>` - Absolute path to saved file

**File naming:** `image-{timestamp}-{random}.{ext}`

**Example:**

```javascript
const imageData = await window.odei.clipboard.readImage();
const base64 = imageData.dataUrl.split(',')[1]; // Strip "data:image/png;base64," prefix
const result = await window.odei.saveImage(base64, 'png');
console.log('Image saved to:', result.path);
// Output: /tmp/odei-images/image-1698765432000-a1b2c3d4.png
```

---

### Token Counting

Count tokens using tiktoken (OpenAI's tokenizer).

#### `window.odei.tiktoken.encode(text)`

Encode text and return token count.

**Parameters:**

- `text` (string) - Text to tokenize

**Returns:** `Promise<number>` - Token count

**Fallback:** If tiktoken fails, uses heuristic: `Math.max(1, Math.round(text.length / 4))`

**Example:**

```javascript
const tokens = await window.odei.tiktoken.encode('Hello, world!');
console.log('Token count:', tokens);
```

---

#### `window.odei.tiktoken.setModel(model)`

Set the tokenizer model (changes encoding rules).

**Parameters:**

- `model` (string) - Model name (e.g., 'gpt-4', 'gpt-3.5-turbo')

**Returns:** `Promise<{ok: boolean, model?: string, error?: string}>`

**Default:** 'gpt-4'

**Example:**

```javascript
const result = await window.odei.tiktoken.setModel('gpt-4');
if (result.ok) {
  console.log('Tokenizer set to:', result.model);
}
```

---

### Companion Terminal

Manage companion terminal instances (secondary terminals for agents).

#### `window.odei.companion.start(agent, provider)`

Start a companion terminal for an agent.

**Parameters:**

- `agent` (string) - Agent identifier
- `provider` (string) - Provider name (default: 'claude')

**Returns:** `Promise<{success: boolean, error?: string}>`

**Example:**

```javascript
const result = await window.odei.companion.start('discuss', 'claude');
if (result.success) {
  console.log('Companion started for discuss');
}
```

---

#### `window.odei.companion.write(agent, data)`

Write data to a companion terminal's stdin.

**Parameters:**

- `agent` (string) - Agent identifier
- `data` (string) - Input data

**Returns:** `Promise<{success: boolean, error?: string}>`

**Example:**

```javascript
await window.odei.companion.write('discuss', 'Hello\n');
```

---

#### `window.odei.companion.resize(agent, cols, rows)`

Resize a companion terminal.

**Parameters:**

- `agent` (string) - Agent identifier
- `cols` (number) - New column count
- `rows` (number) - New row count

**Returns:** `Promise<{success: boolean, error?: string}>`

**Example:**

```javascript
await window.odei.companion.resize('discuss', 120, 40);
```

---

#### `window.odei.companion.kill(agent)`

Kill a companion terminal process.

**Parameters:**

- `agent` (string) - Agent identifier

**Returns:** `Promise<{success: boolean, error?: string}>`

**Example:**

```javascript
await window.odei.companion.kill('discuss');
```

---

#### `window.odei.companion.onOutput(callback)`

Subscribe to companion terminal output.

**Parameters:**

- `callback` (function) - `(agent: string, data: string) => void`

**Returns:** `() => void` - Cleanup function

**Example:**

```javascript
const cleanup = window.odei.companion.onOutput((agent, data) => {
  console.log(`[Companion ${agent}]:`, data);
});
```

---

#### `window.odei.companion.onExit(callback)`

Subscribe to companion terminal exit events.

**Parameters:**

- `callback` (function) - `(agent: string, exitData: {code: number, signal?: string}) => void`

**Returns:** `() => void` - Cleanup function

**Example:**

```javascript
const cleanup = window.odei.companion.onExit((agent, exitData) => {
  console.log(`Companion ${agent} exited with code ${exitData.code}`);
});
```

---

### Lifecycle

Signal renderer readiness to main process.

#### `window.odei.signalReady()`

Signal that the renderer is ready to receive events.

**Important:** Call this once during app initialization. The main process waits for this signal before initializing agents.

**Returns:** void

**Example:**

```javascript
// In your app's initialization code:
window.odei.signalReady();
```

---

## Legacy API: window.electronAPI.\* (Deprecated)

Backward-compatible alias for older code. All methods map to `window.odei.*` equivalents.

**Deprecated methods:**

- `window.electronAPI.getAgentOutput(agent)` - Use `window.odei.terminal.onData()`
- `window.electronAPI.sendToAgent(agent, data)` - Use `window.odei.terminal.write()`
- `window.electronAPI.resizeAgent(agent, cols, rows)` - Use `window.odei.terminal.resize()`
- `window.electronAPI.restartAgent(agent)` - Use `window.odei.terminal.restart()`
- `window.electronAPI.isAgentRunning(agent)` - No direct replacement (check health)
- `window.electronAPI.getHealth()` - Use `window.odei.health.get()`
- `window.electronAPI.callMCPTool(...)` - Use `window.odei.data.get()`
- `window.electronAPI.onAgentOutput(callback)` - Use `window.odei.onAgentOutput()`
- `window.electronAPI.onAgentError(callback)` - Use `window.odei.onAgentError()`
- `window.electronAPI.onHealthUpdate(callback)` - Use `window.odei.onHealthUpdate()`
- `window.electronAPI.onAgentEvent(callback)` - Use `window.odei.onAgentEvent()`
- `window.electronAPI.onContextAdd(callback)` - Use `window.odei.onContextAdd()`
- `window.electronAPI.openExternal(url)` - Use `window.odei.links.openExternal()`
- `window.electronAPI.clipboard.*` - Use `window.odei.clipboard.*`

**Migration:** Replace all `window.electronAPI.*` calls with `window.odei.*` equivalents.

---

## Complete Workflows

### Image Paste Workflow

Complete example of handling image paste from clipboard:

```javascript
// Set up paste handler
window.odei.ipc.on('menu:paste', async () => {
  const formats = await window.odei.clipboard.availableFormats();

  if (formats.hasImage) {
    // Read image from clipboard
    const imageData = await window.odei.clipboard.readImage();
    console.log(`Pasted image: ${imageData.width}x${imageData.height}, ${imageData.sizeKB}KB`);

    // Extract base64 data (remove data URL prefix)
    const base64 = imageData.dataUrl.split(',')[1];

    // Save to temporary file
    const result = await window.odei.saveImage(base64, 'png');
    console.log('Image saved to:', result.path);

    // Use the saved image path
    handleImageFile(result.path);
  } else if (formats.hasText) {
    // Fallback to text paste
    const text = await window.odei.clipboard.readText();
    console.log('Pasted text:', text);
    handleTextPaste(text);
  }
});
```

---

### Text Copy Workflow

Complete example of handling text copy to clipboard:

```javascript
// Set up copy handler
window.odei.ipc.on('menu:copy', async () => {
  // Get selected text from your UI
  const selectedText = getSelectedTextFromUI(); // Your implementation

  if (!selectedText) {
    console.log('Nothing selected to copy');
    return;
  }

  // Copy to clipboard
  const result = await window.odei.clipboard.writeText(selectedText);

  if (result.success) {
    console.log(`Copied ${result.size} bytes to clipboard`);
    showCopyConfirmation(); // Your UI feedback
  } else {
    console.error('Copy failed:', result.error);
    showCopyError(result.error); // Your error handling
  }
});
```

---

### MCP Tool Call Workflow

Complete example of calling an MCP tool and handling results:

```javascript
async function fetchMemories(query) {
  try {
    // Call MCP tool
    const result = await window.odei.data.get(
      'odei-neo4j', // Server name
      'odei.neo4j.memory.recall.v1', // Tool name
      { query, limit: 10 } // Tool parameters
    );

    console.log('Memories retrieved:', result);
    return result;
  } catch (error) {
    console.error('Failed to fetch memories:', error);
    throw error;
  }
}

// Usage:
const memories = await fetchMemories('Recent conversations about ODEI');
```

---

### Agent Terminal Workflow

Complete example of interacting with an agent terminal:

```javascript
// Subscribe to output
const cleanup = window.odei.terminal.onData('discuss', (data) => {
  // Update terminal UI with new data
  appendToTerminal('discuss', data);
});

// Send input to agent
function sendToDiscuss(input) {
  window.odei.terminal.write('discuss', input + '\n');
}

// Handle terminal resize
function handleResize(cols, rows) {
  window.odei.terminal.resize('discuss', cols, rows);
}

// Restart agent on error
async function restartDiscuss() {
  console.log('Restarting discuss agent...');
  window.odei.terminal.restart('discuss');

  // Wait for restart
  await new Promise((resolve) => setTimeout(resolve, 1000));
  console.log('Discuss agent restarted');
}

// Cleanup on component unmount
onUnmount(() => {
  cleanup();
});
```

---

### Health Monitoring Workflow

Complete example of monitoring MCP server health:

```javascript
// Subscribe to health updates
const cleanup = window.odei.onHealthUpdate((health) => {
  console.log('Health status:', health.status);

  // Check each server
  for (const [serverName, serverHealth] of Object.entries(health.servers)) {
    if (serverHealth.status !== 'healthy') {
      console.warn(`${serverName} is ${serverHealth.status}`);
      showHealthWarning(serverName, serverHealth);
    }
  }

  // Update UI
  updateHealthIndicator(health);
});

// Get initial health snapshot
async function initializeHealth() {
  const health = await window.odei.health.get();
  updateHealthIndicator(health);
}

// Cleanup on unmount
onUnmount(() => {
  cleanup();
});
```

---

## Security Notes

### Context Isolation

- **Enabled:** Context isolation is enabled in all renderer processes
- **Node Integration:** Disabled - renderer cannot access Node.js APIs
- **Sandbox:** Enabled - renderer runs in a sandboxed environment
- **Web Security:** Enabled - enforces same-origin policy

### IPC Validation

All IPC handlers validate inputs using Zod schemas before processing. Invalid inputs are rejected and logged.

### Navigation Protection

- New windows are denied via `setWindowOpenHandler()`
- In-app navigation is prevented via `will-navigate` event
- External links require domain allowlist or user confirmation

### CSP (Content Security Policy)

Defined in HTML meta tag:

```html
<meta
  http-equiv="Content-Security-Policy"
  content="default-src 'self';
               script-src 'self';
               style-src 'self' 'unsafe-inline';
               img-src 'self' data:;"
/>
```

### Clipboard Limits

- Text: Maximum 100MB
- Images: No hard limit, but large images may cause performance issues

### Keyboard Interception

Cmd+C/Ctrl+C and Cmd+V/Ctrl+V are intercepted in `before-input-event` to prevent default browser behavior and enable custom copy/paste logic.

---

## Summary of All IPC Handlers

### Total: 28 IPC Handlers

**Agent Management (6):**

1. `agent:get-output`
2. `agent:send-input`
3. `agent:resize`
4. `agent:kill`
5. `agent:restart`
6. `agent:is-running`

**Companion Terminal (5):** 7. `companion:start` 8. `companion:write` 9. `companion:resize` 10. `companion:kill` 11. `companion:output` (event) 12. `companion:exit` (event)

**Clipboard (4):** 13. `clipboard:write` 14. `clipboard:read` 15. `clipboard:readImage` 16. `clipboard:availableFormats`

**Observability Events (6):** 17. `agent:output` (event) 18. `agent:error` (event) 19. `agent:event` (event) 20. `health:update` (event) 21. `agent:context-usage` (event) 22. `agent:context-add` (event)

**Menu Events (2):** 23. `menu:copy` (event) 24. `menu:paste` (event)

**Other Operations (5):** 25. `health:get` 26. `mcp:call-tool` 27. `open-external` 28. `image:save` 29. `tiktoken:encode` 30. `tiktoken:set-model` 31. `renderer:ready`

**Total: 31 handlers/events**

---

## Revision History

- **2025-10-25:** Initial documentation based on IPC audit findings
  - Documented all 31 IPC handlers/events
  - Added previously undocumented menu events (`menu:copy`, `menu:paste`)
  - Added previously undocumented image save API (`image:save`)
  - Added complete workflow examples
  - Added security notes and validation details
