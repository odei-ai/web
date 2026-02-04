# odei-apple MCP Server

Swift EventKit-based MCP server that mediates all Apple Calendar access for ODEI. The Electron frontend and Claude Code agents must call these tools via MCP; no renderer code may touch EventKit directly.

## Quick start (macOS)

```bash
cd servers/odei-apple
swift build -c release
.build/release/odei-apple <<<'{"jsonrpc":"2.0","id":1,"method":"initialize","params":{}}'
```

The binary requests calendar permission on first run. Confirm the macOS TCC prompt so that read/write tools can operate.

## Environment

| Variable                | Default   | Description                                    |
| ----------------------- | --------- | ---------------------------------------------- |
| `ODEI_APPLE_WRITE_MODE` | `confirm` | Write policy: `deny`, `confirm`, or `trusted`. |

`confirm` returns diffs and requires a follow-up call with `confirm:true` and `dryRun:false` before persisting changes. `deny` makes all mutating tools fail. `trusted` executes immediately while still returning audit payloads.

## JSON-RPC examples

```bash
# Health
printf '%s\n' '{"jsonrpc":"2.0","id":1,"method":"call_tool","params":{"name":"odei.apple.health.v1","args":{}}}' \
  | .build/release/odei-apple

# List calendars
printf '%s\n' '{"jsonrpc":"2.0","id":2,"method":"call_tool","params":{"name":"odei.apple.listCalendars.v1","args":{}}}' \
  | .build/release/odei-apple

# Dry-run event creation
printf '%s\n' '{"jsonrpc":"2.0","id":3,"method":"call_tool","params":{"name":"odei.apple.createEvent.v1","args":{"title":"Deep Work","start":"2025-10-10T09:00:00+03:00","end":"2025-10-10T11:00:00+03:00","calendarId":"primary","dryRun":true}}}' \
  | .build/release/odei-apple
```

On non-macOS platforms every tool responds with `{ "error": { "code": 1001, "message": "UNSUPPORTED_PLATFORM" } }`.

## Tool surface

| Tool                            | Behaviour                                                                      |
| ------------------------------- | ------------------------------------------------------------------------------ |
| `odei.apple.health.v1`          | Returns status + TCC authorization state.                                      |
| `odei.apple.listCalendars.v1`   | Lists calendars with modifiability flags.                                      |
| `odei.apple.calendar.window.v1` | Fetches events within a window (recurrence expanded).                          |
| `odei.apple.getEvent.v1`        | Fetches a single event by identifier.                                          |
| `odei.apple.createEvent.v1`     | Create event (dry-run by default, confirm required in `confirm` mode).         |
| `odei.apple.updateEvent.v1`     | Patch event fields.                                                            |
| `odei.apple.deleteEvent.v1`     | Delete event (`scope`: this/future/all).                                       |
| `odei.apple.moveEvent.v1`       | Move event to another calendar.                                                |
| `odei.apple.createCalendar.v1`  | Create an ODEI-owned calendar.                                                 |
| `odei.apple.deleteCalendar.v1`  | Delete calendar (confirm required).                                            |
| `odei.apple.watch.v1`           | Placeholder subscription hook (no-op for now, returns `{ "status": "noop" }`). |

All write operations honour `dryRun`/`confirm` semantics and redact personally identifiable data from logs.

## Claude Code configuration

```json
{
  "mcpServers": {
    "odei-apple": {
      "command": "./servers/odei-apple/.build/release/odei-apple",
      "args": [],
      "cwd": ".",
      "env": {
        "ODEI_APPLE_WRITE_MODE": "confirm"
      }
    }
  }
}
```

## Notes

- The helper must run on macOS 13+ with EventKit access. On Windows/Linux the server compiles but responds with `UNSUPPORTED_PLATFORM`.
- The binary is designed to be bundled inside the Electron app (`extraResources`).
- Logs stream to stdout/stderr and should be captured by the host process. Personally identifiable information is masked before logging.
- Undo semantics for trusted mode are stubbed for now; future iterations should persist snapshots per event in a secure store.
