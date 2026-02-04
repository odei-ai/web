# Agent Security & Isolation

## Overview

Each ODEI agent (Discuss, Decisions, Execute, Mind) is isolated in its own workspace directory and communicates with the system **exclusively through MCP tools**. This document describes the security model and isolation mechanisms.

---

## Isolation Mechanisms

### 1. **Working Directory Isolation**

Each agent is spawned by Electron with a specific `cwd` (current working directory):

```javascript
// electron/agent-manager.js
pty.spawn(claudeBin, claudeFlags, {
  cwd: path.join(agentsDir, agentName), // e.g., /Users/ai/ODEI/agents/discuss
  // ...
});
```

**Result**: Agent sees only its own workspace by default.

### 2. **`.claudeignore` Files**

Each agent directory contains a `.claudeignore` file that explicitly blocks access to:

- `../../src/` — Application UI code
- `../../electron/` — Electron main process code
- `../../node_modules/` — Dependencies
- `../other-agents/` — Other agent workspaces
- `../../.env` — Sensitive configuration

**Location**: `/Users/ai/ODEI/agents/{agentName}/.claudeignore`

### 3. **System Prompt Restrictions**

Each agent's `.claude/CLAUDE.md` contains explicit security rules:

```markdown
## File Access & Security Rules

### ❌ FORBIDDEN Access

- Reading/writing files in `../../src/` (application code)
- Accessing other agent directories
- Running `git status` (shows paths outside workspace)
- Modifying `.env` or sensitive files

### ✅ ALLOWED Access

- Read/write files in your workspace only
- Use MCP tools (ONLY way to access data)
```

**Location**: `/Users/ai/ODEI/agents/{agentName}/.claude/CLAUDE.md`

---

## MCP-Only Data Access

### Core Principle

**Agents NEVER access:**

- Neo4j database directly
- SQLite database directly
- File system outside workspace
- Other agents' files

**Agents ALWAYS use MCP tools:**

- `odei.neo4j.*` — Graph database operations
- `odei.history.*` — Conversation history
- `odei.apple.*` — Calendar/Reminders

### Example: Discuss Agent

```markdown
❌ WRONG:
const neo4j = require('neo4j-driver');
const session = driver.session();

✅ CORRECT:
Use MCP tool: odei.neo4j.memory_recall.v1
```

---

## Agent-Specific Permissions

| Agent         | Allowed MCP Tools                                                     | Purpose                               |
| ------------- | --------------------------------------------------------------------- | ------------------------------------- |
| **Discuss**   | `memory.recall`, `goals.hierarchy`, `calendar.window`, `threads.*`    | Read-only context, record discussions |
| **Decisions** | `decisions.create`, `patterns.analyze`, `goals.hierarchy`             | Record decisions with ROI             |
| **Execute**   | `decisions.pending`, `tasks.create`, `calendar.create` (with confirm) | Create tasks and calendar events      |
| **Mind**      | `patterns.analyze`, `insights.create`, `memory.query`                 | Analyze patterns, record insights     |

---

## Security Guarantees

### ✅ What IS Protected

1. **Application Code**: Agents cannot read/modify UI or Electron code
2. **Database Integrity**: All writes go through validated MCP tools with schemas
3. **Agent Isolation**: Agents cannot interfere with each other
4. **User Confirmation**: Calendar writes require explicit `confirm: true`
5. **Secrets**: `.env` files are blocked from agent access

### ⚠️ What Requires Vigilance

1. **Git Commands**: `git status` shows repo-wide paths (agents instructed not to use it)
2. **Relative Paths**: Agents could theoretically use `../../src/`, but `.claudeignore` blocks it
3. **MCP Server Security**: MCP servers must validate all inputs (Zod schemas in place)

---

## Testing Isolation

To verify agent isolation is working:

1. **Start agent in Electron app** (via terminal UI)
2. **Try forbidden commands**:
   ```bash
   cat ../../src/index.html          # Should be blocked by .claudeignore
   cd ../..                          # Should be warned against in prompt
   git status                        # Should not be used (per instructions)
   ```
3. **Verify MCP-only access**:

   ```bash
   # ✅ This should work:
   Use tool: odei.history.threads_create.v1

   # ❌ This should fail:
   const sqlite = require('better-sqlite3');
   ```

---

## Maintenance

### Adding New Agents

When adding a new agent, ensure:

1. **Create workspace directory**: `/Users/ai/ODEI/agents/{new-agent}/`
2. **Copy `.claudeignore`**: Block `../../src/`, `../../electron/`, other agents
3. **Add security section to `.claude/CLAUDE.md`**: Document forbidden/allowed access
4. **Update `electron/agent-manager.js`**: Add agent to startup array
5. **Document MCP permissions**: List allowed MCP tools in this file

### Reviewing Changes

Before merging changes to agent configurations:

1. **Verify `.claudeignore` is present** and up-to-date
2. **Check `.claude/CLAUDE.md`** has security rules section
3. **Confirm MCP-only access** is enforced in system prompt
4. **Test in Electron app** to ensure isolation works

---

## Threat Model

### In Scope

- **Accidental code modification**: Agent unintentionally edits application files
- **Inter-agent interference**: One agent reads/modifies another agent's state
- **Database bypass**: Agent tries to access Neo4j/SQLite directly
- **Secrets exposure**: Agent reads `.env` or credentials

### Out of Scope (System-Level Attacks)

- **Privilege escalation**: Agent breaks out of Node.js/Electron sandbox (OS-level security)
- **Network attacks**: Agent performs malicious network requests (assumed benign AI behavior)
- **Resource exhaustion**: Agent consumes excessive CPU/memory (monitored by Electron)

---

## References

- **Architecture Document**: [Agents.md](Agents.md)
- **MCP Specifications**: [README odei-neo4j.md](README%20odei-neo4j.md), [README odei-history.md](README%20odei-history.md), [README odei-apple.md](README%20odei-apple.md)
- **Agent Configurations**: `agents/{agentName}/.claude/CLAUDE.md`

---

**Last Updated**: 2025-10-11
**Status**: ✅ Implemented (`.claudeignore` + system prompt restrictions)
