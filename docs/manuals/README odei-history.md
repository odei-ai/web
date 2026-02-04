# odei-history MCP Server

SQLite-backed MCP service that stores raw conversation history for the four ODEI council modules. Canonical facts remain in Neo4j; this service captures drafts, exchanges, and tool logs so they can be curated before promotion. The Electron frontend and Claude Code agents must interact with history exclusively through these MCP tools.

## Quick start

```bash
cd servers/odei-history
cp .env.example .env           # adjust if needed
npm install
npm run build
npm run migrate                # applies migrations/0001_init.sql
npm start                      # serves JSON-RPC over stdio
```

During development you can run `npm run dev` to execute the TypeScript entry point directly (helpful when piping JSON manually).

## Environment

| Variable               | Required | Default                 | Purpose                               |
| ---------------------- | -------- | ----------------------- | ------------------------------------- |
| `ODEI_HISTORY_DB_PATH` | No       | `./data/history.sqlite` | Location of the SQLite database file. |

Secrets are not required. Store the database file in a secure directory; do not commit it to the repository.

## JSON-RPC usage

All communication uses JSON-RPC 2.0 over stdio. Example session:

```bash
printf '%s\n' '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{}}' \
  | node dist/index.js
```

Example tool calls (after build):

```bash
# create a thread
printf '%s\n' '{"jsonrpc":"2.0","id":2,"method":"call_tool","params":{"name":"odei.history.createThread.v1","args":{"module":"discuss","title":"Strategy sync"}}}' \
  | node dist/index.js

# append a message
printf '%s\n' '{"jsonrpc":"2.0","id":3,"method":"call_tool","params":{"name":"odei.history.appendMessage.v1","args":{"threadId":"<thread-id>","module":"discuss","role":"assistant","author":"odei-agent","body":"Hello","contentType":"text/markdown"}}}' \
  | node dist/index.js

# fetch the thread (paginated)
printf '%s\n' '{"jsonrpc":"2.0","id":4,"method":"call_tool","params":{"name":"odei.history.getThread.v1","args":{"threadId":"<thread-id>"}}}' \
  | node dist/index.js
```

Successful responses include `result`, while validation failures surface as `error` objects with `code:422` (`MCP_VALIDATION_FAILED`).

## Tool surface

| Tool                            | Description                                                        |
| ------------------------------- | ------------------------------------------------------------------ |
| `odei.history.createThread.v1`  | Create a thread record for a module.                               |
| `odei.history.appendMessage.v1` | Append a message to an existing thread (assigns sequential `seq`). |
| `odei.history.listThreads.v1`   | List recent threads for a module with cursor pagination.           |
| `odei.history.getThread.v1`     | Fetch a thread with messages (reverse pagination via `beforeSeq`). |
| `odei.history.search.v1`        | LIKE-based search across titles and message bodies.                |

All payloads are validated with Zod; module and role values must match the enumerations defined in `schema.md`.

## Schema & migrations

The authoritative schema lives in [`schema.md`](./schema.md). Migration `0001_init.sql` applies the tables, indexes, pragmas, and triggers described there. Future migrations must be appended numerically and recorded in the `migrations` table via `npm run migrate`.

## Claude Code configuration

Add the server to Claude Code's MCP settings using:

```json
{
  "mcpServers": {
    "odei-history": {
      "command": "node",
      "args": ["./servers/odei-history/dist/index.js"],
      "cwd": ".",
      "env": {
        "ODEI_HISTORY_DB_PATH": "./servers/odei-history/data/history.sqlite"
      }
    }
  }
}
```

## Notes & policies

- Raw conversation logs stay in SQLite and never enter Neo4j directly.
- All IPC payloads use strict Zod validation; invalid modules/roles or malformed metadata return `MCP_VALIDATION_FAILED`.
- `content_hash` is SHA-256 of `content_type:body` and assists with deduplication heuristics.
- The migrations script is idempotent; running it multiple times is safe.

For a combined local environment, see `../dev-all.sh` (launches all MCP servers together).
