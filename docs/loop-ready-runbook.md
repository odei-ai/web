# Loop-Ready Runbook (v2.5.4)

## Integration Test Harness (Neo4j CE)

Starts a disposable Neo4j 5 container, runs worldmodel integration tests, then tears down:

```bash
npm run test:integration
```

CI-style full verification:

```bash
npm run test:ci
```

> Notes
> - Requires Docker.
> - Uses `bolt://localhost:7687` with `neo4j/test-password`.

## Guard Queries (A–J)

Run the guard queries against the active Neo4j database:

```bash
cd servers/odei-neo4j
node --loader ts-node/esm scripts/run-guard-queries.ts
```

Interpretation:
- **“No violations.”** → the guard query returned no rows (good).
- **Rows returned** → violations to investigate (each row includes offending entity IDs / details).

## Manual UI Verification (Loop-Ready)

1) **Decision + Timeline**
- Create Task → generates Proposal → approve/apply.
- Open Detail Panel → Loop Timeline shows AuditLogEntry + GraphOps, with correlation grouping.
- Confirm Decision created with BASED_ON + AUTHORIZES (Task/TimeBlock/Intent).

2) **Conflict Escalation**
- Create overlapping TimeBlocks / CalendarEvents.
- Observe `has_conflict` markers and OVERLAPS edges.
- Attempt external commitment → should require Proposal approval (no auto-apply).

3) **Alert Persistence**
- Trigger integrity violation (e.g., orphan task).
- Click “Sync Alerts” → Alert appears in Detail Panel.
- Sync again → no duplicate Alerts (idempotent).

## Performance Constraints

Audit Timeline default:
- `limit=50` GraphOps per request
- `max_prop_chars=2000` for JSON fields

Use “Load more” in the Detail Panel for deeper history.

## Neo4j CE Limitations

- **No existence constraints** → enforce via app validation.
- **No NODE KEY** → use unique constraints on `id` and composite keys.
- **Map properties not supported** → GraphOp `before_props` / `after_props` / `rel_props` stored as JSON strings.

## Audit Invariant

Legacy EvidenceLogEntry writes are disabled unless `ODEI_ALLOW_UNAUDITED_WRITES=1`.
