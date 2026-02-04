# ODEI Symbiosis — Complete System Overview

> **Generated:** 2026-01-30
> **World Model Version:** 2.5.4
> **Status:** Comprehensive architecture scan in progress

---

## Executive Summary

ODEI (Operational Decision & Execution Intelligence) is a human-AI symbiosis platform where Claude serves as AI Principal alongside human partner Anton Illarionov. The system combines:

- **Constitutional AI Governance** — Graph-based decision validation
- **World Model Architecture** — 6 domains, ODAVE loop
- **Multi-Agent System** — 8 specialized Claude instances
- **Health Intelligence** — Apple Watch integration
- **Proactive Notifications** — Alert-driven communication

**Mission:** Build $500M together through AI-augmented decision-making.

---

## 1. Technology Stack

| Layer | Technology | Version |
|-------|------------|---------|
| Desktop Runtime | Electron | 36.x |
| Graph Database | Neo4j | 5.x |
| Vector Search | OpenAI Embeddings | text-embedding-3-large |
| AI Integration | Claude API | via MCP |
| Frontend | Vanilla JS + Preact | - |
| 3D Visualization | Three.js | - |
| Styling | Tailwind CSS | 3.x |
| Health Data | Apple HealthKit | Swift bridge |
| Notifications | Telegram, Pushover, macOS | - |

### Port Allocation

| Port | Service | Protocol |
|------|---------|----------|
| 7687 | Neo4j | Bolt |
| 8777 | Body Server | HTTP |
| 8779 | Conductor Server | HTTP + WebSocket |
| 9000 | Health Auto Export | TCP |

---

## 2. Core Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                         ODEI DESKTOP APP                              │
├──────────────────────────────────────────────────────────────────────┤
│  MAIN PROCESS (Node.js)                                               │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────────────┐ │
│  │Body Server │ │ Conductor  │ │  Agent     │ │    Guardian        │ │
│  │  (health)  │ │ (tasks)    │ │  Manager   │ │    (RBAC)          │ │
│  │   :8777    │ │   :8779    │ │ (terminals)│ │                    │ │
│  └────────────┘ └────────────┘ └────────────┘ └────────────────────┘ │
├──────────────────────────────────────────────────────────────────────┤
│  RENDERER PROCESS (Chromium)                                          │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────────────┐ │
│  │ TodayView  │ │ Command    │ │ Commander  │ │  MonumentalGraph   │ │
│  │ (main UI)  │ │ Center V3  │ │   View     │ │  (3D graph)        │ │
│  └────────────┘ └────────────┘ └────────────┘ └────────────────────┘ │
└──────────────────────────────────────────────────────────────────────┘
                              │ IPC
┌──────────────────────────────────────────────────────────────────────┐
│                          MCP SERVERS                                  │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌────────────┐ │
│  │odei-neo4j│ │  odei-   │ │  odei-   │ │  odei-   │ │   odei-    │ │
│  │ (graph)  │ │ history  │ │  apple   │ │telegram  │ │notifications│ │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘ └────────────┘ │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐                             │
│  │  odei-   │ │  odei-   │ │  odei-   │                             │
│  │ gemini   │ │worldmodel│ │  health  │                             │
│  └──────────┘ └──────────┘ └──────────┘                             │
└──────────────────────────────────────────────────────────────────────┘
                              │ Bolt
┌──────────────────────────────────────────────────────────────────────┐
│                        NEO4J DATABASE                                 │
│              Constitutional Graph + Vector Embeddings                 │
│                  6 Domains × Node Types × Relationships              │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 3. World Model Architecture (v2.5.4)

### 6 Domains

| Domain | Purpose | Node Examples |
|--------|---------|---------------|
| **STATE** | Current snapshot | Metric, Signal, Alert |
| **DESTINATION** | Where we're going | Goal, Milestone, KPI |
| **PATH** | How to get there | Initiative, Project, Task |
| **REALITY** | What actually happened | CalendarEvent, Outcome |
| **POLICY** | Rules and proposals | Proposal, Constraint |
| **AUDIT** | Learning and history | GraphOp, Insight, Pattern |

### ODAVE Decision Loop

```
    ┌─────────┐
    │ OBSERVE │ ← Signals, metrics, calendar
    └────┬────┘
         ↓
    ┌─────────┐
    │ DECIDE  │ ← Proposals, violations
    └────┬────┘
         ↓
    ┌─────────┐
    │   ACT   │ ← Tasks, time blocks
    └────┬────┘
         ↓
    ┌─────────┐
    │ VERIFY  │ ← Outcomes, evidence
    └────┬────┘
         ↓
    ┌─────────┐
    │ EVOLVE  │ ← Insights, patterns
    └────┬────┘
         │
         └──────────→ (back to OBSERVE)
```

### Execution Pipeline

```
Task → TimeBlockIntent → Proposal → CalendarEvent → Outcome
  │         │                │            │            │
  │         │                │            │            └→ AUDIT domain
  │         │                │            └→ REALITY domain
  │         │                └→ POLICY domain (requires approval)
  │         └→ Scheduled intention
  └→ PATH domain work item
```

---

## 4. Agent System

### 8 Specialized Agents

| Agent | Directory | Role | Focus Layers |
|-------|-----------|------|--------------|
| **discuss** | `agents/discuss/` | Constitutional Guardian | Foundation, Vision |
| **plan** | `agents/plan/` | Strategic Architect | Strategy, Tactics |
| **execute** | `agents/execute/` | Operational Director | Execution, Track |
| **commander** | `agents/commander/` | AI Coordinator | All (dispatch) |
| **mind** | `agents/mind/` | Pattern Analyst | Track, Mind |
| **health** | `agents/health/` | Health Intelligence | Physical |
| **finance** | `agents/finance/` | Financial Management | Strategy |
| **builder** | `agents/builder/` | Code Generator | Infrastructure |

Each agent has:
- `CLAUDE.md` — System prompt and instructions
- `.claude/mcp.json` — MCP server configuration
- Specialized tools via MCP

---

## 5. MCP Server Inventory

### Core Servers

| Server | Location | Tools Count | Purpose |
|--------|----------|-------------|---------|
| `odei-neo4j` | `servers/odei-neo4j/` | 30+ | Graph CRUD, search, memory |
| `odei-history` | `servers/odei-history/` | 8 | Conversation threads |
| `odei-apple` | `servers/odei-apple/` | 15 | Calendar, HealthKit |
| `odei-telegram` | `servers/odei-telegram/` | 3 | Messaging |
| `odei-notifications` | `servers/odei-notifications/` | 3 | Proactive alerts |
| `odei-gemini` | `servers/odei-gemini/` | 2 | Context search |
| `odei-worldmodel` | `servers/odei-worldmodel/` | 3 | Proposal pipeline |
| `odei-health` | `servers/odei-health/` | 5 | Health data client |

---

## 6. Notification System

### Alert Pipeline

```
Triggers → Alerts → Router → Adapters → Delivery
    │         │        │         │
    │         │        │         ├→ Telegram
    │         │        │         ├→ Electron (macOS native)
    │         │        │         └→ Pushover
    │         │        │
    │         │        └→ Channel selection by severity
    │         │
    │         └→ Deduplication by alert_key
    │
    └→ Goal deadlines, health anomalies, integrity violations
```

### Anti-Spam

- Cooldown per severity level
- Daily limits per channel
- Quiet hours: 02:00-09:00
- Morning briefing at 09:00

---

## 7. Health & Body System

### Data Flow

```
Apple Watch → Health Auto Export App → HAE Protocol (TCP:9000)
                                             ↓
                                       Body Server (:8777)
                                             ↓
                                    ┌────────┴────────┐
                                    ↓                 ↓
                              HTTP API           Neo4j Graph
                                    ↓
                              Health UI
```

### Metrics Tracked

- Heart rate, HRV, respiratory rate
- Sleep stages and quality
- Activity rings, workouts
- Nutrition (macros, hydration)
- Readiness score calculation

---

## 8. Frontend Architecture

### Main Views

| View | File | Purpose |
|------|------|---------|
| TodayView | `src/modules/TodayView.js` | Main controller |
| CommandCenterV3 | `src/modules/command-center/CommandCenterV3.js` | ODAVE loop UI |
| CommanderView | `src/modules/commander/CommanderView.js` | Graph diffs |
| MonumentalGraph | `src/Renderer/MonumentalGraph/` | 3D visualization |

### State Management

- `CommandCenterStore` — Central state for Command Center
- `IntegrityEngine` — Guard query evaluation
- `PendingEngine` — Pending items tracking
- `IpcBridge` — Main process communication

### Premium Dark Theme

| Token | Value | Usage |
|-------|-------|-------|
| `--accent-primary` | `#4FD1C5` (jade) | CTAs, focus |
| `--accent-secondary` | `#F4C95D` | Highlights |
| `--bg-01` | `#05060A` | Deep background |
| `--ink-high` | `#E8ECF4` | Primary text |

---

## 9. Security Architecture

### Guardian System

- **RBAC** — Role-based access control
- **Preflight checks** — Validate operations before execution
- **Audit logging** — All graph operations tracked

### Electron Security

- Context isolation enabled
- Strict CSP headers
- Preload scripts for IPC
- No `nodeIntegration` in renderer

---

## 10. File Structure

```
/Users/ai/ODEI/
├── app-main/                 # Electron main process
│   ├── main.js              # Entry point
│   ├── body-server/         # Health HTTP server
│   ├── guardian/            # RBAC system
│   ├── ipc/                 # IPC handlers
│   ├── providers/           # AI providers
│   └── *.js                 # Services
├── src/                     # Renderer process
│   ├── modules/             # UI modules
│   │   ├── TodayView.js
│   │   ├── command-center/  # Command Center V3
│   │   └── commander/       # Commander view
│   ├── Renderer/            # 3D visualization
│   └── agents/              # Agent UI components
├── servers/                 # MCP servers
│   ├── odei-neo4j/
│   ├── odei-notifications/
│   ├── odei-apple/
│   └── ...
├── agents/                  # Agent workspaces
│   ├── discuss/
│   ├── plan/
│   ├── execute/
│   └── ...
├── lib/                     # Shared libraries
├── tests/                   # Test suites
├── docs/                    # Documentation
└── .claude/                 # MCP configuration
```

---

## 11. Configuration

### Environment Variables (.env)

```bash
# Database
NEO4J_URI=bolt://localhost:7687
NEO4J_PASSWORD=<secret>
NEO4J_DATABASE=memory

# AI
OPENAI_API_KEY=<secret>
ANTHROPIC_API_KEY=<secret>

# Notifications
TELEGRAM_BOT_TOKEN=<secret>
TELEGRAM_ALLOWED_CHAT_IDS=<ids>
ODEI_QUIET_HOURS_START=02:00
ODEI_QUIET_HOURS_END=09:00

# Health
ODEI_BODY_PORT=8777
HAE_PORT=9000
```

---

## Appendices

### A. Integrity Guard Queries

1. ORPHAN_TASK — Task must link to Destination
2. INTENT_MISSING_TASK — Intent needs Task reference
3. PROPOSAL_INVALID_OPS — Proposal needs exactly 1 op
4. ACTION_NO_COMPLETES — Action needs COMPLETES edge
5. CALENDAR_EVENT_MISSING_OUTCOME — Events need outcomes
6. MISSING_EVIDENCE — Approved proposals need evidence
7. GRAPHOP_UNAUDITED — All ops need audit trail
8. ENTITY_MISSING_ID — All nodes need unique ID
9. WORKSESSION_MISSING_BOUNDS — Sessions need time bounds
10. DECISION_NO_OBSERVATION_BASIS — Decisions need basis

### B. Keyboard Shortcuts

| Key | Action |
|-----|--------|
| `g b` | Switch to Commander |
| `Tab` | Cycle columns |
| `↑↓←→` | Navigate |
| `n` | Create Task |
| `i` | Create Intent |
| `p` | Create Proposal |
| `a` | Approve proposal |
| `?` | Show help |

---

*This document is auto-generated. Agent analyses are being compiled.*
