# ODEI Notification and Alerting Architecture Review

**Date:** 2026-02-02
**Author:** Claude (AI Principal - Infrastructure Developer mode)
**Status:** Research and Documentation

---

## Executive Summary

The ODEI notification system has evolved organically across three major components with significant overlap in responsibilities. This document analyzes the current state, identifies gaps and redundancies, and proposes a unified architecture.

**Key Finding:** Three systems handle notifications with duplicated logic for cooldowns, quiet hours, and severity handling. Consolidation could reduce ~40% of notification-related code while improving reliability.

---

## Current State Architecture

### System Components

```
+----------------------------------+
|         NOTIFICATION SOURCES      |
+----------------------------------+
         |              |
         v              v
+----------------+  +---------------+
| odei-worldmodel|  | odei-neo4j    |
| Alert nodes    |  | Task/Goal     |
| (Alert:raised) |  | (deadlines)   |
+-------+--------+  +-------+-------+
        |                   |
        +---------+---------+
                  |
                  v
+------------------------------------------+
|     1. odei-notifications (MCP Server)   |
|  - TypeScript, runs on interval          |
|  - Trigger evaluation (health, workload) |
|  - Alert sync with Neo4j                 |
|  - Multi-channel dispatch                |
|  - Morning briefing                      |
+------------------------------------------+
                  |
                  v
+------------------------------------------+
|     2. AI Conductor (Electron Main)      |
|  - JavaScript, event-driven              |
|  - Physical Layer task creation          |
|  - Native macOS notifications            |
|  - Telegram fallback                     |
|  - Work cycle orchestration              |
+------------------------------------------+
                  |
                  v
+------------------------------------------+
|     3. Messenger Daemon (Script)         |
|  - JavaScript, background daemon         |
|  - Telegram polling/bidirectional        |
|  - Alert monitoring (Neo4j)              |
|  - Claude response generation            |
|  - Message routing                       |
+------------------------------------------+
                  |
                  v
+----------------------------------+
|       DELIVERY CHANNELS          |
+----------------------------------+
| - Telegram (odei-telegram MCP)   |
| - Electron/macOS native          |
| - Pushover (critical only)       |
+----------------------------------+
```

### 1. odei-notifications (MCP Server)

**Location:** `/Users/ai/ODEI/servers/odei-notifications/`

**Purpose:** Scheduled proactive notification system with threshold-based alerting.

**Capabilities:**
- Evaluates health triggers (readiness, HRV, sleep debt, stress)
- Evaluates workload triggers (capacity, overdue tasks, deadline risk)
- Syncs alerts to Neo4j worldmodel
- Routes notifications based on severity and quiet hours
- Sends morning briefing (once per day at configured time)
- Supports Telegram, Electron IPC, and Pushover channels

**Key Files:**
- `src/triggers.ts` - Alert trigger evaluation logic
- `src/router.ts` - Notification routing with cooldowns/limits
- `src/notifications.ts` - Main notification check orchestration
- `src/adapters/` - Channel-specific sending implementations
- `config/thresholds.json` - Health/workload thresholds

**State Management:**
- Local file: `~/.odei/notify-state.json`
- Neo4j: Alert nodes with notification metadata
- Log: `~/.odei/notify-log.jsonl`

**Scheduling:** Either via internal interval (`ODEI_NOTIFICATIONS_INTERVAL_MINUTES`) or external launchd.

---

### 2. AI Conductor (Electron Main Process)

**Location:** `/Users/ai/ODEI/app-main/ai-conductor.js`

**Purpose:** Autonomous AI work orchestration with human notification for Physical Layer tasks.

**Capabilities:**
- Wake triggers (launchd, webhook, telegram, manual)
- Context loading from Neo4j (goals, tasks, health)
- Work decision making (capacity, readiness, time of day)
- Headless Claude execution for autonomous work
- Physical Layer task creation and assignment
- Native macOS notifications (primary)
- Telegram notifications (backup)

**Key Features:**
- Evening critic protection (20:00-02:00 deferral)
- Workload capacity checking (>85% blocks new work)
- Readiness score integration (<40% triggers deferral)
- Task assignment via Neo4j `task.create.v1`

**State Management:**
- In-memory: `this.state` object
- Neo4j: Task nodes with `physical_layer` tag

**Status:** Currently DISABLED (requires `ODEI_CONDUCTOR_ENABLED=true`).

---

### 3. Messenger Daemon (Background Script)

**Location:** `/Users/ai/ODEI/scripts/odei-messenger-daemon.js`

**Purpose:** Persistent Telegram integration for bidirectional communication.

**Capabilities:**
- Background polling for Telegram messages (every 10s)
- Neo4j alert monitoring (every 60s)
- Proactive alert notifications with cooldown/limits
- Claude response generation for incoming messages
- Quiet hours respecting (02:00-09:00)
- State persistence across restarts

**Key Features:**
- Daily notification limits per severity
- Cooldown windows (30m critical, 60m high, 180m medium, 360m low)
- Severity escalation detection
- Heartbeat logging (5 min intervals)

**CLI Modes:**
- `--daemon` - Continuous polling loop
- `--check` - Single check cycle
- `--status` - Report current state
- `--send <msg>` - Direct message sending

**State Management:**
- Local file: `~/ODEI/logs/messenger-daemon-state.json`
- Log file: `~/ODEI/logs/messenger-daemon.log`

---

## Overlap Analysis

### Duplicated Functionality

| Capability | odei-notifications | AI Conductor | Messenger Daemon |
|------------|-------------------|--------------|------------------|
| Telegram sending | Yes (via MCP) | Yes (via MCP) | Yes (via MCP) |
| Quiet hours | 02:00-09:00 | 20:00-02:00 | 02:00-09:00 |
| Cooldown logic | Per-severity | N/A | Per-severity |
| Daily limits | Per-severity | N/A | Per-severity |
| Severity normalization | Yes | Yes | Yes |
| Alert monitoring | Yes (Neo4j) | No | Yes (Neo4j) |
| macOS native | Via Electron IPC | Direct | No |
| Health integration | Yes | Yes | No |
| Workload integration | Yes | Yes | No |

**Severity Normalization:** Identical function exists in three files:
- `servers/odei-notifications/src/router.ts:normalizeSeverity()`
- `scripts/odei-messenger-daemon.js:normalizeSeverity()`
- `app-main/ai-conductor.js` (inline)

**Cooldown Logic:** Similar but not identical implementations:
- odei-notifications: `shouldNotify()` with escalation detection
- Messenger Daemon: `shouldNotify()` with escalation detection
- Conductor: No cooldown (task-based, not alert-based)

**Quiet Hours:** Two different definitions:
- Notifications/Daemon: 02:00-09:00 (sleep protection)
- Conductor: 20:00-02:00 (evening critic protection)

---

## Gaps in the Notification Pipeline

### 1. Missing Alert Sources
- **Calendar conflicts** - Only notifications server checks this
- **Health emergencies** - No real-time push notification on sudden HRV crash
- **Goal completion** - No celebration/acknowledgment notifications
- **System errors** - No unified error alerting

### 2. Channel Limitations
- **No SMS fallback** - Critical alerts rely on Telegram availability
- **No email digest** - Daily/weekly summary not available
- **No web push** - If Electron not running, limited options

### 3. Coordination Gaps
- **No unified queue** - Three systems may send duplicate alerts
- **No acknowledgment sync** - Dismissing in one system doesn't update others
- **No delivery confirmation** - No verification messages were received

### 4. State Fragmentation
- Three separate state files with different formats
- No shared "notification sent" log accessible to all systems
- Alert state in Neo4j may drift from local state files

### 5. Missing Features
- **Snooze functionality** - No way to temporarily mute specific alerts
- **Priority escalation** - Time-based severity increase not implemented
- **User preferences** - No per-channel preference management
- **Rate limiting** - Per-channel rate limits not enforced

---

## Proposed Unified Architecture

### Core Principles

1. **Single Source of Truth:** Neo4j Alert nodes own notification state
2. **Event-Driven:** Use MQTT/Redis pub-sub for real-time coordination
3. **Channel Abstraction:** Unified adapter interface for all channels
4. **Centralized Routing:** One routing engine with pluggable rules

### Proposed Architecture

```
+----------------------------------+
|         ALERT SOURCES            |
+----------------------------------+
| - Health service (real-time)     |
| - Workload service (Neo4j)       |
| - Calendar service (conflicts)   |
| - System monitoring              |
+----------------------------------+
              |
              v
+----------------------------------+
|     UNIFIED ALERT BUS            |
|  (Redis/MQTT pub-sub)            |
+----------------------------------+
              |
              v
+------------------------------------------+
|     NOTIFICATION ENGINE (TypeScript)     |
|  Location: servers/odei-notifications    |
|                                          |
|  Components:                             |
|  - AlertIngester: validates, dedupes     |
|  - StateManager: Neo4j + local cache     |
|  - Router: severity, time, preferences   |
|  - Scheduler: cooldowns, limits, snooze  |
|  - Dispatcher: channel adapters          |
+------------------------------------------+
              |
    +---------+---------+
    |         |         |
    v         v         v
+-------+ +-------+ +--------+
|Telegram| |macOS  | |Pushover|
|Adapter | |Adapter| |Adapter |
+-------+ +-------+ +--------+
              |
              v
+----------------------------------+
|     DELIVERY CONFIRMATION        |
|  (webhook/polling)               |
+----------------------------------+
              |
              v
+----------------------------------+
|     NEO4J (Alert Nodes)          |
|  - notification_state            |
|  - last_delivered_at             |
|  - delivery_channel              |
|  - acknowledged_at               |
+----------------------------------+
```

### Component Responsibilities

**Notification Engine (servers/odei-notifications):**
- Absorb Messenger Daemon alert monitoring
- Absorb Conductor notification logic
- Expose MCP tools for all notification operations
- Own the complete notification lifecycle

**AI Conductor (app-main/ai-conductor.js):**
- Focus on work orchestration only
- Delegate notifications to engine via MCP
- Remove duplicate Telegram/macOS code

**Messenger Daemon (scripts/odei-messenger-daemon.js):**
- Convert to pure Telegram polling relay
- Forward messages to notification engine
- Remove alert monitoring (handled by engine)
- Keep Claude response generation

---

## Migration Plan

### Phase 1: State Consolidation (1-2 days)

**Goal:** Single source of truth for notification state.

**Tasks:**
1. Create `NotificationState` schema in Neo4j
2. Migrate `~/.odei/notify-state.json` to Neo4j
3. Migrate `logs/messenger-daemon-state.json` to Neo4j
4. Update odei-notifications to read/write Neo4j directly
5. Add fallback to local state when Neo4j unavailable

**Deliverables:**
- New Neo4j constraint: `CREATE CONSTRAINT ON (n:NotificationState) ASSERT n.alert_key IS UNIQUE`
- Migration script for existing state files

### Phase 2: Routing Consolidation (2-3 days)

**Goal:** Single routing engine with shared utilities.

**Tasks:**
1. Create `@odei/notifications-core` shared package
   - `normalizeSeverity()`
   - `isWithinQuietHours()`
   - `shouldNotify()`
   - `applyNotificationState()`
2. Refactor odei-notifications to use shared package
3. Remove duplicate code from Messenger Daemon
4. Update Conductor to use shared routing via MCP

**Deliverables:**
- `packages/notifications-core/` with shared types and utilities
- Updated imports in all three systems

### Phase 3: Channel Abstraction (1-2 days)

**Goal:** Unified channel interface.

**Tasks:**
1. Define `NotificationChannel` interface
2. Move adapters to `@odei/notifications-core`
3. Add channel registration mechanism
4. Add delivery confirmation hooks

**Interface:**
```typescript
interface NotificationChannel {
  id: string;
  name: string;
  send(payload: NotificationPayload): Promise<DeliveryResult>;
  checkDelivery?(messageId: string): Promise<DeliveryStatus>;
  isAvailable(): Promise<boolean>;
}
```

### Phase 4: Daemon Refactoring (1-2 days)

**Goal:** Messenger Daemon becomes thin relay.

**Tasks:**
1. Remove alert monitoring from Daemon
2. Add MCP call to `odei.notifications.run_check.v1`
3. Keep only Telegram polling and response logic
4. Add incoming message forwarding to engine

### Phase 5: Conductor Cleanup (1 day)

**Goal:** Conductor delegates to notification engine.

**Tasks:**
1. Remove `sendAppleNotification()`
2. Remove `sendTelegramNotification()`
3. Replace with MCP call: `odei.notifications.send.v1`
4. Keep task creation logic only

### Phase 6: Event Bus Integration (2-3 days)

**Goal:** Real-time alert propagation.

**Tasks:**
1. Add Redis/MQTT to infrastructure
2. Publish alerts on creation/update
3. Subscribe notification engine to alert events
4. Add real-time health emergency detection

---

## Priority Improvements

### P0: Critical (This Week)

1. **Fix duplicate notifications**
   - Add `notification_id` to track across systems
   - Query Neo4j before sending to check recent sends
   - Estimated effort: 4 hours

2. **Unified quiet hours**
   - Consolidate to single definition (02:00-09:00 for sleep, 20:00-22:00 warning)
   - Move config to single `.env` variable
   - Estimated effort: 2 hours

### P1: High Priority (Next Sprint)

3. **Delivery confirmation**
   - Track message IDs from Telegram
   - Add retry logic for failed deliveries
   - Estimated effort: 6 hours

4. **Alert acknowledgment sync**
   - When dismissed in one channel, update Neo4j
   - Propagate to other systems
   - Estimated effort: 8 hours

5. **Snooze functionality**
   - Add `/snooze <alert> <duration>` command
   - Store snooze end time in Neo4j
   - Estimated effort: 4 hours

### P2: Medium Priority (Next Month)

6. **Email digest channel**
   - Daily summary of alerts and health
   - Weekly goal progress report
   - Estimated effort: 8 hours

7. **Rate limiting per channel**
   - Max 1 Telegram per minute
   - Max 3 Pushover per hour
   - Estimated effort: 4 hours

8. **User preferences**
   - Channel preferences per alert type
   - DND schedule beyond quiet hours
   - Estimated effort: 6 hours

### P3: Low Priority (Backlog)

9. **SMS fallback channel**
   - Twilio integration for critical alerts
   - Only when Telegram/Pushover fail
   - Estimated effort: 8 hours

10. **Web push notifications**
    - Service worker for ODEI web view
    - PWA support
    - Estimated effort: 12 hours

---

## Configuration Consolidation

### Current Environment Variables (Fragmented)

```bash
# odei-notifications
ODEI_QUIET_HOURS_START=02:00
ODEI_QUIET_HOURS_END=09:00
ODEI_BRIEFING_TIME=09:00
ODEI_NOTIFICATIONS_INTERVAL_MINUTES=15
ODEI_TELEGRAM_CHAT_ID=
ODEI_TELEGRAM_DRY_RUN=false
PUSHOVER_USER_KEY=
PUSHOVER_APP_TOKEN=

# AI Conductor
ODEI_CONDUCTOR_ENABLED=false
ODEI_USE_APPLE_NOTIFICATIONS=true
ODEI_USE_TELEGRAM_NOTIFICATIONS=true

# Messenger Daemon
ODEI_MESSENGER_POLL_INTERVAL=10000
ODEI_MESSENGER_ALERT_INTERVAL=60000
ODEI_MESSENGER_RESPOND=true
```

### Proposed Unified Configuration

```bash
# Core notification settings (shared)
ODEI_NOTIFICATION_QUIET_START=02:00
ODEI_NOTIFICATION_QUIET_END=09:00
ODEI_NOTIFICATION_EVENING_START=20:00  # Evening critic warning
ODEI_NOTIFICATION_EVENING_END=02:00    # Before quiet hours

# Channel enablement (shared)
ODEI_CHANNEL_TELEGRAM_ENABLED=true
ODEI_CHANNEL_TELEGRAM_CHAT_ID=
ODEI_CHANNEL_TELEGRAM_DRY_RUN=false
ODEI_CHANNEL_MACOS_ENABLED=true
ODEI_CHANNEL_PUSHOVER_ENABLED=false
ODEI_CHANNEL_PUSHOVER_USER_KEY=
ODEI_CHANNEL_PUSHOVER_APP_TOKEN=

# Scheduling (notification engine only)
ODEI_NOTIFICATION_CHECK_INTERVAL=15
ODEI_NOTIFICATION_BRIEFING_TIME=09:00

# Messenger daemon (polling only)
ODEI_TELEGRAM_POLL_INTERVAL=10000
ODEI_TELEGRAM_RESPOND_ENABLED=true

# Conductor (work orchestration only)
ODEI_CONDUCTOR_ENABLED=false
```

---

## Metrics and Monitoring

### Current State

- **No unified metrics** - Each system logs independently
- **No delivery rate tracking** - Unknown message success rate
- **No latency monitoring** - Unknown time from trigger to delivery

### Proposed Metrics

1. **Notification throughput**
   - Alerts triggered per hour
   - Notifications sent per channel per hour
   - Delivery success rate per channel

2. **Latency metrics**
   - Alert detection to notification sent
   - Notification sent to delivery confirmed

3. **User engagement**
   - Acknowledgment rate
   - Average time to acknowledge
   - Snooze frequency

4. **System health**
   - Neo4j connection availability
   - Channel availability (Telegram API, etc.)
   - State file sync status

---

## Conclusion

The current ODEI notification architecture has grown organically to serve different use cases, resulting in significant code duplication and fragmented state management. The proposed unified architecture consolidates these systems while preserving their distinct responsibilities:

1. **Notification Engine** - All alert evaluation, routing, and delivery
2. **AI Conductor** - Work orchestration only
3. **Messenger Daemon** - Telegram relay and Claude responses

The migration can be executed incrementally over 2-3 weeks, with each phase delivering immediate value. The P0 improvements (duplicate prevention and unified quiet hours) should be prioritized to prevent notification fatigue.

---

## Appendix: File Inventory

| File | Lines | Purpose | Action |
|------|-------|---------|--------|
| `servers/odei-notifications/src/notifications.ts` | 636 | Main orchestration | Keep, extend |
| `servers/odei-notifications/src/router.ts` | 173 | Routing logic | Extract to shared |
| `servers/odei-notifications/src/triggers.ts` | 275 | Alert triggers | Keep |
| `servers/odei-notifications/src/adapters/telegram.ts` | 35 | Telegram send | Move to shared |
| `servers/odei-notifications/src/adapters/electron.ts` | 88 | Electron IPC | Move to shared |
| `servers/odei-notifications/src/adapters/pushover.ts` | 77 | Pushover send | Move to shared |
| `app-main/ai-conductor.js` | 755 | Work orchestration | Remove notification code |
| `scripts/odei-messenger-daemon.js` | 1009 | Telegram daemon | Simplify to relay |
| `servers/odei-worldmodel/src/tools/alerts.ts` | 186 | Alert CRUD | Keep |

**Estimated total notification-related code:** ~3,234 lines
**Estimated after consolidation:** ~1,900 lines (41% reduction)
