# ODEI Calendar Integration Architecture

## Overview

Integration of Apple Calendar (via `odei-apple` MCP server) with ODEI agents to provide real-time workload assessment and schedule management.

## Architecture Principle

**Minimalist approach**: Only agents that directly interact with time/schedule have calendar access.

```
┌─────────────────────────────────────────┐
│          odei-apple MCP Server          │
│    Swift EventKit → JSON-RPC Bridge     │
└─────────────────────────────────────────┘
              ↓                    ↓
    [Discuss: READ]      [Execute: READ + WRITE]
              ↓                    ↓
        workload_assess     schedule optimization
```

## Agent Access Matrix

| Agent         | Calendar Access | Tools                                        | Purpose                                             |
| ------------- | --------------- | -------------------------------------------- | --------------------------------------------------- |
| **Discuss**   | READ only       | `window.v1`, `listCalendars.v1`, `health.v1` | Workload assessment before adding Goals             |
| **Execute**   | READ + WRITE    | All 6 tools (+ create/update/delete)         | Schedule optimization, TimeBlock management         |
| **Decisions** | ❌ None         | —                                            | Gets workload data via Neo4j or inter-agent queries |
| **Mind**      | ❌ None         | —                                            | Analyzes WorkSession/TimeBlock nodes from Neo4j     |

## Discuss Agent: Workload Assessment

### Use Case

Before accepting new Goals/Commitments, Discuss checks **real capacity**:

### Workflow

1. **Query Neo4j Tasks:**

   ```cypher
   MATCH (t:Task)
   WHERE t.status IN ['todo', 'in_progress']
   RETURN sum(t.effortHours) as taskHours
   ```

2. **Query Apple Calendar:**

   ```javascript
   odei.apple.calendar.window.v1({
     from: '2025-10-24T00:00:00Z',
     to: '2025-10-31T00:00:00Z',
   });
   // Sum event durations
   ```

3. **Calculate Total Capacity:**

   ```
   Available hours = 7 days × 10hrs = 70hrs
   Scheduled hours = taskHours + calendarHours
   Capacity % = (Scheduled / Available) × 100
   ```

4. **Apply Thresholds:**
   - `<50%`: Underutilized — can suggest more Goals
   - `50-85%`: Optimal — proceed with light flag
   - `>85%`: Overload — block new commitments, require trade-offs

### Example Response

```
Our schedule is at 87% capacity this week.
- Neo4j Tasks: 35hrs (4 active tasks)
- Calendar Events: 26hrs (7 meetings)
= 61hrs / 70hrs available

Adding [new Goal] requires 10hrs/week → 101% capacity.
Recommend: Defer to next week OR deprioritize [specific task].
```

## Execute Agent: Schedule Optimization

### Use Case

Propose optimal daily/weekly schedules matching Tony's energy patterns.

### Workflow

1. **Load Active Tasks from Neo4j:**

   ```cypher
   MATCH (t:Task)
   WHERE t.status = 'todo' OR t.status = 'in_progress'
   RETURN t ORDER BY t.priority DESC
   ```

2. **Query Calendar for Free Slots:**

   ```javascript
   odei.apple.calendar.window.v1({
     from: '2025-10-25T09:00:00Z',
     to: '2025-10-25T19:00:00Z',
   });
   // Identify gaps between meetings
   ```

3. **Match Tasks to Energy Patterns:**
   - **09:00-14:00 (Peak)**: Complex/creative tasks
   - **14:00-18:00 (Execution)**: Implementation work
   - **18:00-20:00 (Admin)**: Low-cognitive tasks

4. **Propose TimeBlocks with DryRun:**

   ```javascript
   odei.apple.createEvent.v1({
     title: 'Task: Architect Mind agent',
     start: '2025-10-25T09:00:00Z',
     end: '2025-10-25T11:30:00Z',
     dryRun: true, // Preview only
   });
   ```

5. **After Human Approval:**

   ```javascript
   odei.apple.createEvent.v1({
     // same params
     confirm: true, // Actually write to calendar
   });
   ```

6. **Create Neo4j TimeBlock:**
   ```javascript
   odei.neo4j.time_block.create.v1({
     data: {
       start: '2025-10-25T09:00:00Z',
       end: '2025-10-25T11:30:00Z',
       calendarRef: {
         provider: 'apple',
         externalId: '[event-id]',
       },
     },
   });
   ```

## Calendar Tools Reference

### Read Tools (Discuss + Execute)

**`odei.apple.calendar.window.v1`**

```javascript
{
  from: "2025-10-24T00:00:00Z",  // ISO 8601 UTC
  to: "2025-10-31T00:00:00Z",
  calendarIds?: ["id1", "id2"]   // Optional filter
}
// Returns: events[] with id, title, start, end, isAllDay
```

**`odei.apple.listCalendars.v1`**

```javascript
{
}
// Returns: calendars[] with id, title, color, type, isDefault
```

**`odei.apple.health.v1`**

```javascript
{
}
// Returns: { status: "ok", authorizationStatus: "authorized" }
```

### Write Tools (Execute only)

**`odei.apple.createEvent.v1`**

```javascript
{
  title: "Event title",
  start: "2025-10-25T09:00:00Z",
  end: "2025-10-25T11:00:00Z",
  location?: "Optional location",
  notes?: "Optional notes",
  url?: "https://...",
  isAllDay?: false,
  calendarId?: "specific-calendar-id",
  dryRun?: true,    // Preview only, don't create
  confirm?: true    // Actually create (requires dryRun=false)
}
```

**`odei.apple.updateEvent.v1`**

```javascript
{
  eventId: "event-uuid",
  updates: {
    title?: "New title",
    start?: "2025-10-25T10:00:00Z",
    // ... any fields to update
  },
  dryRun?: true,
  confirm?: true
}
```

**`odei.apple.deleteEvent.v1`**

```javascript
{
  eventId: "event-uuid",
  scope?: "this" | "future",  // Delete single or recurring
  dryRun?: true,
  confirm?: true
}
```

## Integration with Neo4j Graph

### TimeBlock → Calendar Linkage

Execute creates **bidirectional mapping**:

**Neo4j TimeBlock node:**

```cypher
CREATE (tb:TimeBlock {
  id: "tb_123",
  title: "Task: Architect Mind agent",
  start: "2025-10-25T09:00:00Z",
  end: "2025-10-25T11:30:00Z",
  calendarRef: {
    provider: "apple",
    externalId: "APPLE-EVENT-UUID"
  },
  status: "active"
})
```

**Apple Calendar event:**

- Contains reference to TimeBlock ID in notes or custom field
- Allows sync: if calendar event updated manually → can detect drift

### Sync Strategy

**Calendar → Neo4j (Read sync):**

- Execute periodically queries `calendar.window.v1` for upcoming week
- Compares with TimeBlock nodes in Neo4j
- Flags mismatches: "Calendar shows 2hrs meeting not in Neo4j"

**Neo4j → Calendar (Write sync):**

- All TimeBlock writes go through `createEvent.v1` with `confirm: true`
- Neo4j TimeBlock creation ALWAYS paired with calendar write
- Orphaned Neo4j TimeBlocks (no calendarRef) = not yet scheduled

## Authorization & Security

### EventKit Permissions (macOS)

**First run:**

```bash
swift run odei-apple
# macOS prompts: "Allow odei-apple to access Calendar?"
# User must approve
```

**Check status:**

```javascript
odei.apple.health.v1();
// Returns: { authorizationStatus: "authorized" | "denied" | "notDetermined" }
```

**If denied:**

- User must manually enable in System Settings → Privacy & Security → Calendars

### Confirmation Pattern (Prevent Accidental Writes)

**All write operations use two-step confirmation:**

1. **DryRun (Preview):**

   ```javascript
   createEvent({ ..., dryRun: true })
   // Returns: "Would create event '[title]' from [start] to [end]"
   ```

2. **Confirm (Actual Write):**
   ```javascript
   createEvent({ ..., confirm: true })
   // Actually writes to calendar
   ```

**Agents MUST:**

- Always preview with `dryRun: true` first
- Present to human: "Proposed schedule: [details]"
- Only write with `confirm: true` after explicit approval

## Performance Considerations

### MCP Server Process

**Persistent process** (not killed between requests):

- Agents spawn `swift run -c release` once
- Process stays alive, handles multiple JSON-RPC requests
- Faster than spawning per request (Swift has 2-3s startup cost)

**Health monitoring:**

- `health.v1` called every 5 seconds
- If fails → restart process automatically

### Query Optimization

**Narrow time windows:**

```javascript
// Good: 7-day window
calendar.window.v1({ from: today, to: today+7days })

// Bad: 365-day window (slow, high memory)
calendar.window.v1({ from: today, to: today+365days })
```

**Calendar filtering:**

```javascript
// Only query work calendar
calendar.window.v1({
  calendarIds: ['WORK-CALENDAR-ID'],
});
```

## Troubleshooting

### Calendar Not Authorized

```javascript
odei.apple.health.v1();
// { authorizationStatus: "denied" }

// Fix: System Settings → Privacy & Security → Calendars → Enable odei-apple
```

### Events Not Appearing

```javascript
// Check if calendar ID is correct
odei.apple.listCalendars.v1();
// Verify calendarId in createEvent call matches
```

### Duplicate Events

```javascript
// Use idempotencyKey in TimeBlock creation
odei.neo4j.time_block.create.v1({
  idempotencyKey: 'task_123_2025-10-25_09:00',
});
// Prevents duplicate TimeBlocks for same event
```

## Future Enhancements

### Phase 2: Decisions + Mind Calendar Access

**If metrics show:**

- Decisions frequently requests workload from Discuss (>5x/day)
- Mind cannot achieve accurate velocity predictions without raw calendar

**Then add:**

- Decisions: READ only (workload assessment for ROI)
- Mind: READ only (calendar analytics for pattern detection)

### Phase 3: Bi-directional Sync

**Goal:** Detect external calendar changes

```javascript
// Webhook from Calendar.app (if possible)
// Or polling: every 5 minutes query calendar → compare with Neo4j
```

**Use case:**

- Meeting moved manually in Calendar.app
- Execute detects drift: "TimeBlock tb_123 scheduled 09:00-11:00, but calendar shows 10:00-12:00"
- Propose: "Update Neo4j to match calendar?" or "Revert calendar to Neo4j?"

---

## Summary

**Current state:**

- ✅ Discuss: READ calendar for workload assessment
- ✅ Execute: FULL calendar for schedule management
- ✅ Decisions/Mind: No calendar (use Neo4j + inter-agent queries)

**Benefits:**

- Real-time capacity awareness prevents overcommitment
- Automated schedule optimization respects existing meetings
- Bidirectional Neo4j ↔ Calendar linkage maintains single source of truth

**Architecture principle:**
Minimalist, pragmatic access model — only agents that directly manage time have calendar tools.
