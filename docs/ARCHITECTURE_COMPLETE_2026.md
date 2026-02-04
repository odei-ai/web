# ODEI Symbiosis — Complete System Architecture

> **Generated:** 2026-01-30
> **World Model Version:** 2.5.4
> **Total Code Scanned:** ~100,000+ lines across 500+ files
> **Analysis Agents:** 11 parallel scans

---

## Executive Summary

ODEI (Operational Decision & Execution Intelligence) is a human-AI symbiosis platform where Claude serves as AI Principal alongside human partner Anton Illarionov. The system combines:

- **Constitutional AI Governance** — Graph-based decision validation
- **World Model Architecture** — 6 domains, ODAVE loop
- **Multi-Agent System** — 8 specialized Claude instances
- **Health Intelligence** — Apple Watch integration
- **Proactive Notifications** — Multi-channel alert system

**Mission:** Build $500M together through AI-augmented decision-making.

---

## Table of Contents

1. [Technology Stack](#1-technology-stack)
2. [Electron Main Process](#2-electron-main-process)
3. [MCP Server Ecosystem](#3-mcp-server-ecosystem)
4. [World Model Architecture](#4-world-model-architecture)
5. [Neo4j Graph Schema](#5-neo4j-graph-schema)
6. [Agent System](#6-agent-system)
7. [Frontend Architecture](#7-frontend-architecture)
8. [Health & Body System](#8-health--body-system)
9. [Notification System](#9-notification-system)
10. [Testing Infrastructure](#10-testing-infrastructure)
11. [Build Configuration](#11-build-configuration)
12. [Shared Libraries](#12-shared-libraries)

---

## 1. Technology Stack

| Layer | Technology | Version |
|-------|------------|---------|
| Desktop Runtime | Electron | 33.x |
| Graph Database | Neo4j | 5.x |
| Vector Search | OpenAI Embeddings | text-embedding-3-large |
| AI Integration | Claude API | via MCP |
| Frontend | Vanilla JS + Preact | - |
| 3D Visualization | Three.js | 0.181 |
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

## 2. Electron Main Process

**Location:** `/Users/ai/ODEI/app-main/`
**Lines:** ~8,700 across 20+ modules

### 2.1 Architecture Diagram

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
```

### 2.2 Entry Point (`main.js`)

**Initialization Flow:**

1. **PATH Resolution** (lines 8-25) — Fixes Node.js PATH for GUI apps
2. **GPU Isolation** (lines 51-56) — Limits V8 heap to 512MB to coexist with other Chromium apps
3. **Environment & Config** (lines 58-92) — Loads `.env`, resolves paths
4. **app.whenReady() Lifecycle** (lines 520-716) — Initializes all services

**Critical Services:**

| Service | Purpose | Start Timing |
|---------|---------|--------------|
| HealthProbes | Monitors MCP servers | Before renderer |
| AIConductor | Autonomous work orchestration | Before renderer |
| BodyServer | HTTP: health ingest | Before renderer |
| ConductorServer | WebSocket: agent task dispatch | Before renderer |
| AgentManager | Spawns agent PTY processes | On renderer:ready |

### 2.3 IPC Handlers

**Location:** `/Users/ai/ODEI/app-main/ipc/` (10 modules)

| Module | Key Channels |
|--------|--------------|
| agent-handlers.js | `agent:start`, `agent:kill`, `agent:send-input` |
| mcp-handlers.js | `mcp:call-tool`, `mcp:stream-tool` |
| conductor-handlers.js | `conductor:wake`, `conductor:get-state` |
| backup-handlers.js | `backup:create`, `backup:restore` |
| settings-handlers.js | `settings:get-all`, `settings:save` |

### 2.4 AI Conductor (`ai-conductor.js`)

**Purpose:** Autonomous work orchestration with health-aware task selection

**Wake Flow:**
1. Load fresh state from Neo4j
2. Check health/readiness
3. Decide work (work/defer/assign_physical)
4. Execute work cycle OR assign Physical Layer task
5. Clear working flag, emit 'sleep'

**Decision Rules:**
- Evening Critic: 20:00-02:00 blocks Foundation/Vision changes
- Workload > 85%: Defer work
- Readiness < 40%: Recommend rest

**Headless Execution:**
```javascript
spawn('claude', ['-p', prompt, '--output-format', 'json', '--max-turns', '15'])
```

### 2.5 Conductor Server (`conductor-server.js`)

**Purpose:** WebSocket server for multi-agent task orchestration
**Port:** 8779

**Message Types:**

| Message | Direction | Purpose |
|---------|-----------|---------|
| `hello` | client→server | Handshake |
| `welcome` | server→client | Session setup |
| `assign_task` | client→server | Dispatch task |
| `task_complete` | client→server | Mark done |
| `get_output` | client→server | Fetch buffer |

**Task Lifecycle:**
```
PENDING → ASSIGNED → RUNNING → COMPLETED/FAILED/TIMEOUT
```

---

## 3. MCP Server Ecosystem

**Total Servers:** 11
**Total Tools:** 60+

### 3.1 Server Inventory

| Server | Type | Tools | Purpose |
|--------|------|-------|---------|
| odei-neo4j | Node.js | 32 | Constitutional graph, memory |
| odei-history | Node.js | 9 | Conversation persistence |
| odei-apple | Swift | 13 | Calendar, HealthKit |
| odei-telegram | Node.js | 3 | Messaging |
| odei-notifications | Node.js | 3 | Proactive alerts |
| odei-gemini | Node.js | 2 | Context search |
| odei-worldmodel | Node.js | 7 | Proposal pipeline |
| odei-health | Node.js | 2 | Health data bridge |
| odei-guardian | Node.js | 6 | Trust & verification |
| odei-miro | Node.js | TBD | Miro boards |
| gateway | Node.js | - | API gateway |

### 3.2 odei-neo4j (Primary Graph Server)

**Tools by Category:**

**Layer Management (14 tools):**
- `odei.neo4j.{layer}.create.v1` — Create nodes in layer
- `odei.neo4j.{layer}.list.v1` — List nodes in layer

**Node Operations (4 tools):**
- `odei.neo4j.node.create.v1` — Create typed nodes
- `odei.neo4j.node.update.v1` — Patch nodes
- `odei.neo4j.node.get.v1` — Retrieve with relationships
- `odei.neo4j.node.delete.v1` — Archive or delete

**Search (3 tools):**
- `odei.neo4j.hybrid.search.v1` — Semantic + keyword + graph
- `odei.neo4j.hybrid.plusplus.search.v1` — With temporal decay
- `odei.neo4j.personalized.search.v1` — User-adapted ranking

**Memory (5 tools):**
- `odei.neo4j.memory.retrieve.v1` — Staged retrieval
- `odei.neo4j.memory.coverage.v1` — Gap detection
- `odei.neo4j.temporal.context.v1` — Current context
- `odei.neo4j.temporal.autofix.v1` — Repair inconsistencies

### 3.3 odei-worldmodel (Proposal Pipeline)

**Tools:**
- `odei.worldmodel.create_proposal.v1` — Draft proposal
- `odei.worldmodel.decide_proposal.v1` — Accept/reject
- `odei.worldmodel.apply_proposal.v1` — Execute approved
- `odei.worldmodel.alerts.sync.v1` — Sync violations
- `odei.worldmodel.record_outcome.v1` — Log outcomes

**Pipeline Flow:**
```
Proposal → Validate → Authority → Decision → Apply → Evidence
```

### 3.4 odei-apple (Swift Server)

**Calendar Tools:**
- `odei.apple.listCalendars.v1`
- `odei.apple.calendar.window.v1`
- `odei.apple.createEvent.v1`
- `odei.apple.updateEvent.v1`
- `odei.apple.deleteEvent.v1`

**Health Tools:**
- `odei.apple.health.authorize.v1`
- `odei.apple.health.readiness.v1`
- `odei.apple.health.heartRate.v1`
- `odei.apple.health.hrv.v1`
- `odei.apple.health.sleep.v1`
- `odei.apple.health.workouts.v1`
- `odei.apple.health.vitals.v1`
- `odei.apple.health.activity.v1`

---

## 4. World Model Architecture

### 4.1 Six Domains

| Domain | Purpose | Key Entities |
|--------|---------|--------------|
| **STATE** | Current snapshot | Metric, Signal, Alert |
| **DESTINATION** | Goals & vision | Goal, Milestone, KPI |
| **PATH** | How to get there | Initiative, Project, Task |
| **REALITY** | What happened | CalendarEvent, Outcome |
| **POLICY** | Rules & proposals | Proposal, Constraint |
| **AUDIT** | History & learning | GraphOp, Insight, Pattern |

### 4.2 ODAVE Decision Loop

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

### 4.3 Execution Pipeline

```
Task → TimeBlockIntent → Proposal → CalendarEvent → Outcome
  │         │                │            │            │
  │         │                │            │            └→ AUDIT domain
  │         │                │            └→ REALITY domain
  │         │                └→ POLICY domain (requires approval)
  │         └→ Scheduled intention
  └→ PATH domain work item
```

### 4.4 Authority Engine

**Classification by Risk Tier:**
- LOW: Autonomous execution allowed
- MEDIUM: Requires review
- HIGH: Requires human approval
- CRITICAL: Always requires human

**Loop Phase Requirements:**
- OBSERVE: Read-only operations
- DECIDE: Proposal creation
- ACT: Requires prior DECIDE authorization
- VERIFY: Outcome recording
- EVOLVE: Pattern updates

---

## 5. Neo4j Graph Schema

### 5.1 Node Types by Layer (52 total)

**Foundation (11 types):**
- Value, Principle, Guardrail, Asset, Human, Core, AI, Policy, Partnership, Context, Relationship

**Vision (4 types):**
- Vision, Business, Goal, Season

**Strategy (6 types):**
- Strategy, Objective, KeyResult, Initiative, Milestone, Risk

**Tactics (4 types):**
- Project, Area, System, Process

**Execution (6 types):**
- Decision, Task, TimeBlock, TimeBlockIntent, WorkSession, Action

**Track (12 types):**
- Metric, Observation, Event, Signal, CalendarEvent, CalendarDay, Resource, Outcome, Person, Organization, Belief, Fact

**Mind (7 types):**
- Insight, Pattern, Evidence, Artifact, Experience, Source, Note

**Policy (2 types):**
- Proposal, Alert

**Audit (3 types):**
- AuditLogEntry, GraphOp, ActionOp

### 5.2 Key Relationships (100+ types)

**Foundation Layer:**
- `HOLDS_VALUE`, `FOLLOWS_PRINCIPLE`, `RESPECTS_GUARDRAIL`, `DEFINED_BY`, `PARTNERS_WITH`

**Vision Layer:**
- `ALIGNS_WITH`, `ENABLES`, `HAS_GOAL`, `PURSUES_GOAL`, `HOLDS_EQUITY`, `SERVES`

**Strategy Layer:**
- `HAS_KEY_RESULT`, `MEASURES`, `CONTAINS`, `ADVANCES`, `HAS_MILESTONE`, `HAS_RISK`

**Execution Layer:**
- `SCHEDULED_AS`, `LOGGED_AS`, `BASED_ON`, `AUTHORIZES`

### 5.3 Multi-Label System

```cypher
Node Labels: [:Entity, :Task, :Path, :Intent]
Node Labels: [:Entity, :Fact, :Observation]
Node Labels: [:Entity, :Anchor, :Vision]
```

### 5.4 Provenance Model

All writes require:
```typescript
{
  module: 'discuss' | 'plan' | 'execute' | ...,
  actor: 'human' | 'agent' | 'joint',
  source: string,
  confidence: number
}
```

---

## 6. Agent System

### 6.1 Agent Roster

| Agent | Role | Layer Scope | Key Distinction |
|-------|------|-------------|-----------------|
| **Commander** | AI Principal | All 7 layers | Unified interface, dispatch |
| **Plan** | Strategic Architect | Strategy, Tactics | ROI analysis, decomposition |
| **Execute** | Operational Director | Execution, Track | Energy-aware scheduling |
| **Mind** | Psychological Companion | Track, Mind | Emotional support |
| **Health** | Health Intelligence | Track, Physical | Biometrics, readiness |
| **Finance** | Financial Intelligence | Strategy, Vision | Wealth tracking |
| **Builder** | Infrastructure Developer | Engineering | Code, UI, MCP servers |

### 6.2 Commander Agent (Primary)

**Responsibilities:**
1. Constitutional Guardian — Protect Foundation
2. Strategic Leadership — Vision cascade
3. Operational Judgment — Capacity management
4. Multi-Agent Coordination — Dispatch

**Protection Rules:**
- Evening Critic (20:00-02:00): Flag Vision/Goal changes
- Workload > 85%: Must sacrifice something
- Readiness < 60%: Reduce load

**Session Start Protocol:**
```javascript
odei.neo4j.temporal.context.v1({ timezone: 'Asia/Qatar' })
odei.neo4j.memory.coverage.v1({ layers: ['foundation', 'vision'] })
odei.neo4j.memory.retrieve.v1({ agent: 'commander', tokenBudget: 15000 })
odei.health.state.v1({})
odei.history.threads.list.v1({ module: 'commander', limit: 1 })
```

### 6.3 Execute Agent

**Energy-Aware Scheduling:**

| Time Block | Energy | Best For |
|------------|--------|----------|
| 09:00-12:00 | Peak | Deep work |
| 12:00-14:00 | Dip | Admin, emails |
| 14:00-17:00 | Recovery | Meetings |
| 17:00-20:00 | Second wind | Creative work |
| 20:00+ | Evening | Light only |

**Calendar Sacred Protocol:**
1. Always dry-run first
2. Get explicit consent
3. Only then create
4. Link to graph

### 6.4 Mind Agent

**Core Principles:**
- Empathy first — validate feelings
- User-led pace — let user choose depth
- Non-judgmental curiosity
- Strengths-based approach

**What Mind Is NOT:**
- NOT a licensed clinician
- NOT a diagnostician
- NO medical advice

---

## 7. Frontend Architecture

### 7.1 Module Structure

```
src/modules/
├── TodayView.js              # Main dashboard controller
├── command-center/
│   ├── CommandCenterV3.js    # Loop phase UI
│   ├── CommandCenterStore.js # State management
│   ├── DetailPanel.js        # Node inspector
│   └── KeyboardManager.js    # Shortcuts
├── commander/
│   ├── CommanderView.js      # Decision hub
│   └── GraphDiffView.js      # Diff visualization
├── dashboard/
│   └── LayerGraphLayout.js   # 3D graph
└── IpcBridge.js              # Electron IPC
```

### 7.2 Component Hierarchy

```
renderer.js (App bootstrap)
├─ IpcBridge (Electron IPC)
├─ UIManager (Global orchestration)
├─ TodayView (Main dashboard)
│   ├─ CommandCenterStore (State)
│   │   ├─ IntegrityEngine
│   │   └─ PendingEngine
│   ├─ CommandCenterV3 (Loop UI)
│   ├─ DetailPanel (Inspector)
│   └─ MonumentalGraph (3D)
```

### 7.3 MonumentalGraph (3D Visualization)

**6 Rendering Layers:**
1. Core — Node/edge state
2. Rendering — Three.js scene
3. Policy — Visibility rules
4. Effects — Post-processing
5. Interaction — Mouse/keyboard
6. World Model — Domain coloring

### 7.4 Premium Dark Theme

| Token | Value | Usage |
|-------|-------|-------|
| `--accent-primary` | `#4FD1C5` (jade) | CTAs, focus |
| `--accent-secondary` | `#F4C95D` | Highlights |
| `--bg-01` | `#05060A` | Deep background |
| `--ink-high` | `#E8ECF4` | Primary text |

---

## 8. Health & Body System

### 8.1 Data Flow

```
Apple Watch → Health Auto Export App → HAE Protocol (TCP:9000)
                                             ↓
                                       Body Server (:8777)
                                             ↓
                                    ┌────────┴────────┐
                                    ↓                 ↓
                              HTTP API           Neo4j Graph
```

### 8.2 Body Server Endpoints

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/body/state` | GET | Full state (v1/v2) |
| `/body/event` | POST | Log manual event |
| `/body/meal` | POST | Log meal |
| `/body/water` | POST | Log intake |
| `/body/garmin/sync` | POST | Garmin sync |
| `/body/apple/sync` | POST | Apple sync |
| `/body/recovery/weekly` | GET | Weekly analysis |

### 8.3 Metrics Tracked

**Vitals:**
- Heart rate, HRV, Resting HR
- Blood oxygen, Respiratory rate
- Wrist temperature

**Activity:**
- Steps, Distance, Flights climbed
- Active/Basal energy
- Exercise minutes, Stand hours

**Sleep:**
- Total, Deep, REM, Light, Awake minutes
- Sleep efficiency, Sleep score
- Bedtime, Wake time

### 8.4 Readiness Score Calculation

```
readiness =
  sleepScore × 0.35 +
  hrvFactor × 0.25 +
  restingHRFactor × 0.20 +
  activityBalance × 0.20
```

---

## 9. Notification System

### 9.1 Pipeline

```
Triggers → Alerts → Router → Adapters → Delivery
    │         │        │         │
    │         │        │         ├→ Telegram
    │         │        │         ├→ Electron (macOS)
    │         │        │         └→ Pushover
    │         │        │
    │         │        └→ Channel selection by severity
    │         │
    │         └→ Deduplication by alert_key
    │
    └→ Goal deadlines, health anomalies, integrity violations
```

### 9.2 Trigger Types

| Trigger | Severity | Condition |
|---------|----------|-----------|
| Readiness Low | CRITICAL | < 50% |
| Workload High | CRITICAL | > 85% capacity |
| Overdue Tasks | HIGH | High-priority > 24h |
| Deadline Risk | HIGH | < 7 days, > 5 tasks |
| Sleep Debt | HIGH | > 2 hours |
| Severe Stress | CRITICAL | HR ≥ 80, HRV ≤ 25 |

### 9.3 Anti-Spam Rules

**Cooldown (per severity):**
- CRITICAL: 30 minutes
- HIGH: 60 minutes
- MEDIUM: 180 minutes
- LOW: 360 minutes

**Daily Limits:**
- CRITICAL: 5
- HIGH: 8
- MEDIUM: 3
- LOW: 2

**Quiet Hours:** 22:00-09:00 (CRITICAL bypasses)

---

## 10. Testing Infrastructure

### 10.1 Test Frameworks

| Framework | Purpose | Config |
|-----------|---------|--------|
| Vitest | Unit tests | vitest.config.ts |
| Playwright | E2E tests | playwright.config.ts |
| Custom runners | Neo4j integration | npm scripts |

### 10.2 Test Coverage

| Area | Tests | Coverage |
|------|-------|----------|
| Node Types | 38/38 | 100% |
| Tools | 97+ | Comprehensive |
| Edge Cases | 50+ | Varied |
| Integration | 200+ | Staged |

### 10.3 Test Commands

```bash
npm test                # Vitest run
npm run test:coverage   # Coverage report
npm run test:playwright # E2E tests
npm run test:ci         # Full CI suite
```

---

## 11. Build Configuration

### 11.1 NPM Scripts

| Category | Script | Command |
|----------|--------|---------|
| Build | `build` | Build MCP servers |
| Electron | `start` | Production start |
| Dev | `dev` | CSS + Electron watch |
| CSS | `build:css` | Tailwind build |
| Test | `test:ci` | Full CI suite |

### 11.2 Electron Builder

```json
{
  "appId": "com.odei.app",
  "productName": "ODEI",
  "mac": { "target": "dmg", "arch": "arm64" }
}
```

### 11.3 TypeScript Configuration

- **Target:** ES2022
- **Module:** ES2022
- **Strict:** true
- **Paths:** `@/*`, `@modules/*`, `@types/*`

---

## 12. Shared Libraries

### 12.1 @odei/embeddings

**Purpose:** Embedding generation & quality control

**Exports:**
- `generateEmbedding(text, options)`
- `generateEmbeddingsBatch(texts, options)`
- `validateNodeData(node, nodeType)`
- `assessEmbeddingQuality(embedding, metadata)`
- `createNodeEmbedding(node, nodeType, context)`

### 12.2 @odei/agent-neo4j-mcp

**Modules:**
- `semantic-search.js` — Vector similarity
- `embeddings.js` — Neo4j plugin embeddings
- `graph-algorithms.js` — GDS algorithms
- `hybrid-search.js` — Multi-strategy search
- `apoc-helpers.js` — Traversal utilities

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

### C. Environment Variables

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
ODEI_QUIET_HOURS_START=02:00
ODEI_QUIET_HOURS_END=09:00

# Health
ODEI_BODY_PORT=8777
HAE_PORT=9000
```

---

*Generated by 11 parallel analysis agents scanning the complete ODEI codebase.*
*Total analysis time: ~5 minutes*
*Document version: 2026.01.30*
