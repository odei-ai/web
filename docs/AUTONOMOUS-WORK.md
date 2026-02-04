# ODEI Autonomous Work Infrastructure

## Overview

The AI Conductor enables Claude to work autonomously without waiting for human prompts.

## Architecture

```
                    ┌─────────────────┐
                    │   launchd       │
                    │ (scheduled)     │
                    └────────┬────────┘
                             │ every 30min (09:00-20:00)
                             ▼
                    ┌─────────────────┐
                    │ conductor-wake  │
                    │   .sh script    │
                    └────────┬────────┘
                             │ HTTP POST
                             ▼
┌──────────────────────────────────────────────────────┐
│                   ODEI Electron                       │
│                                                       │
│  ┌──────────────┐       ┌───────────────────┐        │
│  │ Body Server  │──────▶│   AI Conductor    │        │
│  │ :8777        │       │                   │        │
│  └──────────────┘       └─────────┬─────────┘        │
│                                   │                   │
│                                   ▼                   │
│                         ┌───────────────────┐        │
│                         │  Agent Manager    │        │
│                         │ (PTY wrapper)     │        │
│                         └─────────┬─────────┘        │
│                                   │                   │
│                                   ▼                   │
│                         ┌───────────────────┐        │
│                         │   Claude Code     │        │
│                         │ (discuss agent)   │        │
│                         └─────────┬─────────┘        │
│                                   │                   │
│                                   ▼                   │
│                         ┌───────────────────┐        │
│                         │    Neo4j Graph    │        │
│                         │  (tasks, goals)   │        │
│                         └───────────────────┘        │
└──────────────────────────────────────────────────────┘
                             │
                             │ creates Physical Layer task
                             ▼
              ┌──────────────┴──────────────┐
              │                             │
              ▼                             ▼
     ┌─────────────────┐           ┌─────────────────┐
     │ Apple Notif.    │           │    Telegram     │
     │ (primary)       │           │  (backup)       │
     └─────────────────┘           └─────────────────┘
```

## Components

### 1. AI Conductor (`electron/ai-conductor.js`)

Orchestrates autonomous work cycles:

- **wake()** — Triggered by schedule, webhook, or manual
- **decideWork()** — Checks time, health, capacity before proceeding
- **executeWorkCycle()** — Sends autonomous prompt to agent
- **assignPhysicalLayerTask()** — Creates tasks for human execution
- **notifyPhysicalLayer()** — Sends notifications via Apple + Telegram
- **sendAppleNotification()** — Native macOS notification (primary)
- **sendTelegramNotification()** — Telegram message (backup)

### 2. Body Server Endpoints

**Wake Conductor:**

```
POST http://localhost:8777/conductor/wake
Body: { "trigger": "scheduled" | "manual" | "webhook" }
```

**Complete Task:**

```
POST http://localhost:8777/conductor/task-complete
Body: { "taskId": "uuid", "status": "done" | "blocked", "notes": "optional" }
```

### 3. Launch Agent (`scripts/com.odei.conductor.plist`)

Runs every 30 minutes during Doha work hours (09:00-20:00).

### 4. Wake Script (`scripts/conductor-wake.sh`)

Shell script that triggers the conductor via HTTP.

## Installation

### 1. Enable Real Telegram Notifications

Edit `.env`:

```bash
ODEI_TELEGRAM_DRY_RUN=false  # Enable real messages
ODEI_TELEGRAM_CHAT_ID=1437134423  # Anton's chat
```

### 2. Install Launch Agent

```bash
# Copy plist to LaunchAgents
cp /Users/ai/ODEI/scripts/com.odei.conductor.plist ~/Library/LaunchAgents/

# Load the job
launchctl load ~/Library/LaunchAgents/com.odei.conductor.plist

# Verify it's loaded
launchctl list | grep odei
```

### 3. Manual Trigger

```bash
# Via launchctl
launchctl start com.odei.conductor

# Via script
/Users/ai/ODEI/scripts/conductor-wake.sh manual

# Via curl
curl -X POST http://localhost:8777/conductor/wake \
  -H "Content-Type: application/json" \
  -d '{"trigger": "manual"}'
```

## Work Cycle Flow

1. **Wake** — Conductor wakes via trigger
2. **Load State** — Fetch tasks, goals, health from Neo4j
3. **Decide** — Check evening critic, capacity, readiness
4. **Execute** — Send autonomous prompt to Claude agent
5. **Work** — Claude executes tasks from graph
6. **Handoff** — Create Physical Layer tasks when needed
7. **Notify** — Telegram notification to Anton
8. **Sleep** — Wait for next wake trigger

## Evening Critic Protection

The conductor respects evening critic hours (20:00-02:00 Doha):

- Defers strategic work to morning
- Returns `nextWake` time for scheduler
- Allows pure execution tasks if already in progress

## Physical Layer Tasks

When AI creates a task requiring human action:

1. Task created in Neo4j with tag `physical_layer`
2. Notifications sent:
   - **Apple Notification** (primary) — instant, native macOS
   - **Telegram** (backup) — works when ODEI not running
3. Task appears in ODEI Today view with 🧬 PL badge
4. Anton completes via:
   - ODEI UI → `conductor.taskComplete()`
   - HTTP API → `POST /conductor/task-complete`
   - Apple Shortcuts (using HTTP endpoint)

## Monitoring

### Logs

```bash
tail -f /Users/ai/ODEI/logs/conductor.log
tail -f /Users/ai/ODEI/logs/conductor.error.log
```

### State

```bash
curl http://localhost:8777/conductor/state
```

### IPC (from renderer)

```javascript
window.odei.conductor.getState();
window.odei.conductor.wake('manual');
```

## Configuration

| Env Variable                      | Default      | Description                       |
| --------------------------------- | ------------ | --------------------------------- |
| `ODEI_TELEGRAM_DRY_RUN`           | `true`       | Set `false` to send real messages |
| `ODEI_TELEGRAM_CHAT_ID`           | `1437134423` | Anton's Telegram ID               |
| `ODEI_USE_APPLE_NOTIFICATIONS`    | `true`       | Enable native macOS notifications |
| `ODEI_USE_TELEGRAM_NOTIFICATIONS` | `true`       | Enable Telegram notifications     |
| `ODEI_BODY_PORT`                  | `8777`       | Body server port                  |
| `ODEI_BODY_TOKEN`                 | (none)       | Optional auth token               |

## Troubleshooting

### Conductor not waking

1. Check ODEI app is running
2. Check launchd job: `launchctl list | grep odei`
3. Check logs: `tail /Users/ai/ODEI/logs/conductor.log`

### Telegram not sending

1. Verify `ODEI_TELEGRAM_DRY_RUN=false`
2. Check bot token is valid
3. Check chat ID is in allowed list

### Agent not responding

1. Verify agent is running in ODEI UI
2. Check MCP servers are connected
3. Restart agent: `window.odei.agent.restart('discuss')`

## Testing

### Test Notifications

From DevTools console:

```javascript
// Test both Apple + Telegram notifications
window.odei.conductor.testNotification();
```

### Test Task Completion

```bash
curl -X POST http://localhost:8777/conductor/task-complete \
  -H "Content-Type: application/json" \
  -d '{"taskId": "your-task-id", "status": "done", "notes": "Completed via API"}'
```

### Manual Wake

```bash
curl -X POST http://localhost:8777/conductor/wake \
  -H "Content-Type: application/json" \
  -d '{"trigger": "manual"}'
```
