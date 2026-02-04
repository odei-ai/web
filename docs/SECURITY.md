# ODEI Security Guidelines

This document outlines security practices and hardening measures implemented in the ODEI Electron application.

## Overview

ODEI is a multi-agent AI orchestration platform built with Electron. Security is critical given:

- Multi-agent architecture with autonomous operation
- IPC communication between renderer and main process
- File system access for images and backups
- External connections to Neo4j, MCP servers, and APIs
- Terminal emulation with PTY processes

## Content Security Policy (CSP)

**Location:** `/Users/ai/ODEI/src/index.html` lines 8-26

### Current Policy

```
default-src 'self';
script-src 'self' 'unsafe-inline' 'wasm-unsafe-eval' https://esm.sh https://cdn.jsdelivr.net;
style-src 'self' 'unsafe-inline';
img-src 'self' data:;
font-src 'self';
connect-src 'self' https: ws: wss:;
worker-src 'self' blob:;
object-src 'none';
base-uri 'self';
form-action 'self'
```

### Rationale for Unsafe Directives

- **`'unsafe-inline'` in script-src**: Required for Preact JSX runtime and inline event handlers in renderer.js. Alternative: Use nonces or hashes for inline scripts (future improvement).

- **`'wasm-unsafe-eval'` in script-src**: Required for force-graph-3d library (3D physics engine uses WASM). Cannot be removed without replacing the library.

- **`'unsafe-inline'` in style-src**: Required for dynamic styles in ThreeGraphRenderer and DashboardKanbanView. Alternative: Use CSS-in-JS with CSP nonces (future improvement).

- **External CDN (esm.sh, cdn.jsdelivr.net)**: Force-graph-3d and dependencies loaded from CDN. **TODO:** Vendor libraries locally to eliminate external dependencies.

- **`connect-src https: ws: wss:`**: Allows HTTPS connections to Neo4j database, MCP servers, and WebSocket connections. HTTP is explicitly blocked (changed from `http: https:` to `https:` only).

### Security Improvements Made

1. **Removed HTTP from connect-src**: Changed from `http: https:` to `https:` only to prevent insecure connections.
2. **Added comments**: Documented why each unsafe directive is needed for future audits.

### Future Improvements

- [ ] Replace force-graph-3d with CSP-compatible library or vendor WASM modules
- [ ] Implement nonce-based CSP for inline scripts and styles
- [ ] Vendor all external libraries locally (eliminate CDN dependencies)
- [ ] Use `script-src 'self'` without `unsafe-inline` after refactoring

## Path Traversal Protection

**Location:** `/Users/ai/ODEI/electron/main.js` lines 417-483

### Vulnerability

The `image:load` IPC handler accepts a filepath from the renderer and loads image files. Without proper validation, an attacker could:

- Use `../` sequences to escape the allowed directory
- Use null bytes (`\0`) to bypass extension checks
- Create symlinks pointing outside the allowed directory (TOCTOU attack)

### Protection Measures

1. **Pre-validation**: Check for path traversal patterns (`..`, `\0`, absolute paths) BEFORE resolving the path.

2. **Directory containment**: Only allow access to `/tmp/odei-images/` (or OS-specific temp directory).

3. **Basename extraction**: Use `path.basename()` to strip directory components from user input.

4. **Symlink resolution**: Use `fs.realpathSync()` to resolve symlinks atomically.

5. **Post-resolution verification**: Re-check that the resolved path is still within the allowed directory (prevents TOCTOU).

### Code Pattern

```javascript
// ✅ SECURITY: Validate input before processing
if (!filepath || typeof filepath !== 'string') {
  throw new Error('Invalid filepath');
}

// ✅ SECURITY: Check for path traversal patterns BEFORE resolving
if (filepath.includes('..') || filepath.includes('\0') || filepath.startsWith('/')) {
  throw new Error('Invalid filepath: path traversal detected');
}

// ✅ SECURITY: Build target path from safe base (prevents traversal)
const targetPath = path.join(resolvedBase, path.basename(filepath));

// ✅ SECURITY: Resolve symlinks and verify final path
const resolved = fs.realpathSync(normalizedTarget);
if (!resolved.startsWith(normalizedBase)) {
  throw new Error('Access denied: symlink outside allowed directory');
}
```

## Input Validation

**Locations:**

- `/Users/ai/ODEI/electron/preload.js` lines 1-254
- `/Users/ai/ODEI/src/renderer.js` lines 84-111

### IPC Bridge Validation

All IPC calls from renderer to main process are validated in the preload script using helper functions:

1. **`validateAgentName(agent)`**: Validates agent names against a whitelist of allowed agents.
   - Allowed: `['discuss', 'plan', 'execute', 'mind', 'health', 'finance', 'builder', 'research', 'document']`
   - Prevents arbitrary command execution via agent name injection

2. **`validateString(value, name)`**: Ensures value is a string (prevents type confusion attacks).

3. **`validateNumber(value, name, min, max)`**: Validates numeric inputs with range checks.
   - Terminal dimensions: `cols` (1-500), `rows` (1-200)
   - Prevents resource exhaustion attacks

### Renderer Validation

The `window.restartAgent()` function in renderer.js also validates agent names to provide defense-in-depth:

```javascript
// ✅ SECURITY: Validate agent parameter
if (!agent || typeof agent !== 'string') {
  console.error('[ODEIApp] Invalid agent parameter:', agent);
  return;
}

// ✅ SECURITY: Whitelist allowed agent names
  const allowedAgents = [
  'discuss',
  'plan',
  'execute',
  'mind',
  'health',
  'finance',
  'builder',
  'research',
  'document',
];
if (!allowedAgents.includes(agent)) {
  console.error('[ODEIApp] Unauthorized agent name:', agent);
  return;
}
```

### Validation Coverage

All IPC endpoints that accept user input are validated:

- ✅ `agent:start`, `agent:kill`, `agent:restart`, `agent:is-running`
- ✅ `agent:send-input`, `agent:resize`
- ✅ `companion:start`, `companion:write`, `companion:resize`, `companion:kill`
- ✅ `image:load` (path validation)
- ⚠️ `mcp:call-tool` (validated server-side by MCP servers)
- ⚠️ `conductor:*` (trusted internal calls only)

## Hardcoded Paths Removal

**Location:** `/Users/ai/ODEI/electron/agent-manager.js` lines 17-55

### Issues

Original code had hardcoded user-specific paths:

- Claude binary: `/Users/ai/.nvm/versions/node/v22.14.0/bin/claude`
- Node binary: `/Users/ai/.nvm/versions/node/v22.14.0/bin`

These paths:

- Only work on the developer's machine
- Break on other systems with different node installations
- Expose internal directory structure

### Fix

1. **Claude binary**: Use `CLAUDE_BIN` environment variable or let PATH resolve the binary.

   ```javascript
   const claudeBin = this.env.CLAUDE_BIN || (process.platform === 'win32' ? 'claude.cmd' : 'claude');
   ```

2. **Node binary**: Use `NODE_BIN_DIR` environment variable or detect from `process.execPath`.
   ```javascript
   const nodeBinDir = this.env.NODE_BIN_DIR || path.dirname(process.execPath);
   ```

### Configuration

Set environment variables in `.env` or shell:

```bash
export CLAUDE_BIN=/path/to/claude
export NODE_BIN_DIR=/path/to/node/bin
```

## Security Best Practices

### 1. Principle of Least Privilege

- **Preload script**: Only expose necessary APIs to renderer via `contextBridge`
- **Agent whitelist**: Only allow known agent names (no arbitrary process execution)
- **File system access**: Restrict to specific directories (`/tmp/odei-images/`)

### 2. Defense in Depth

- **Multiple validation layers**: Validate in renderer, preload, and main process
- **Path checks**: Pre-validation + post-resolution verification
- **Type checks**: Validate all input types before processing

### 3. Secure Defaults

- **CSP**: Strict policy with documented exceptions
- **IPC**: No arbitrary command execution (whitelist approach)
- **Paths**: Use environment variables or PATH resolution (no hardcoded paths)

### 4. Error Handling

- **Never leak internal paths**: Use generic error messages in responses
- **Log detailed errors**: Keep detailed logs in main process for debugging
- **Fail securely**: Reject invalid input rather than attempting to sanitize

## Threat Model

### Threats Mitigated

1. **Path Traversal (CWE-22)**: ✅ Fixed via validation + containment checks
2. **Command Injection (CWE-77)**: ✅ Mitigated via agent name whitelist
3. **XSS (CWE-79)**: ⚠️ Partially mitigated via CSP (still allows `unsafe-inline`)
4. **TOCTOU Race (CWE-367)**: ✅ Fixed via atomic symlink resolution
5. **Resource Exhaustion (CWE-400)**: ✅ Mitigated via input range validation

### Residual Risks

1. **CSP unsafe directives**: `unsafe-inline` and `wasm-unsafe-eval` reduce XSS protection
   - **Mitigation**: Use Electron's process isolation + context isolation
   - **Future fix**: Refactor to use nonce-based CSP

2. **External CDN dependencies**: esm.sh and cdn.jsdelivr.net introduce supply chain risk
   - **Mitigation**: CSP restricts to HTTPS only
   - **Future fix**: Vendor all libraries locally

3. **MCP server trust**: MCP servers run with full Node.js privileges
   - **Mitigation**: Only load trusted MCP servers from `.claude/mcp.json`
   - **Future fix**: Sandbox MCP servers or run in separate processes

## Security Review Checklist

When adding new features:

- [ ] Validate all IPC inputs (type, range, whitelist)
- [ ] Never trust renderer-provided paths (use containment checks)
- [ ] Use environment variables for binaries (no hardcoded paths)
- [ ] Test with malicious inputs (path traversal, command injection)
- [ ] Review CSP impact (does feature require unsafe directives?)
- [ ] Document security rationale in code comments
- [ ] Add tests for security edge cases

## Reporting Security Issues

Security vulnerabilities should be reported to:

- **Email**: [security contact - TODO]
- **Scope**: Electron app, MCP servers, agent orchestration
- **Response time**: 48 hours for critical issues

Do not open public GitHub issues for security vulnerabilities.

## References

- [Electron Security Checklist](https://www.electronjs.org/docs/latest/tutorial/security)
- [Content Security Policy (CSP)](https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP)
- [Path Traversal (OWASP)](https://owasp.org/www-community/attacks/Path_Traversal)
- [Node.js Security Best Practices](https://nodejs.org/en/docs/guides/security/)

---

**Last Updated:** 2025-12-13
**Security Audit:** Hardening sprint (CSP, path traversal, input validation)
