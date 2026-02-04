# ADR-010: SQLite for Conversation History Persistence

## Status
Accepted

## Context

Claude agents have no native memory across sessions. Each new conversation starts from scratch, requiring:

- **Context reloading:** Re-query Neo4j for Values, Goals, Projects
- **Emotional continuity:** Lose track of user mood, unresolved tensions
- **Promise tracking:** Forget commitments made in previous sessions
- **Pattern detection:** Cannot analyze conversation trends over time

ODEI needs to remember:
- What was discussed in last session
- User's emotional state (tired, frustrated, excited)
- Outstanding questions/promises
- Conversation patterns (topics, frequency, duration)

### Problem

How do we persist conversation history in a way that enables:
1. **Session continuity:** Claude remembers last conversation
2. **Searchable history:** Find past discussions by topic
3. **Analytics:** Detect patterns (e.g., "Anton asks about goals every Monday")
4. **Module isolation:** Discuss conversations separate from Execute conversations
5. **Performance:** Query last 25 messages in <10ms

### Constraints

1. **Privacy:** Conversations stored locally (no cloud sync)
2. **Reliability:** Must survive app crashes/restarts
3. **Queryability:** Support full-text search on message content
4. **Structure:** Thread-based model (conversations have continuity)
5. **Modularity:** Each agent has separate message history
6. **Backup:** Export/import for disaster recovery

## Decision

**Use SQLite for local conversation persistence with thread-based model.**

SQLite provides:
1. **Zero-configuration:** Embedded database (no separate service)
2. **ACID transactions:** Reliable writes even during crashes
3. **Full-text search:** FTS5 extension for content search
4. **Portability:** Single file, easy to backup/transfer
5. **Performance:** 10,000+ reads/sec on modern hardware
6. **Node.js integration:** `better-sqlite3` (synchronous, fast)

### Schema Design

```sql
-- Threads (conversation containers)
CREATE TABLE threads (
  id TEXT PRIMARY KEY,             -- UUID
  module TEXT NOT NULL,            -- 'discuss' | 'plan' | 'execute' | 'mind' | ...
  title TEXT,                      -- Optional user-provided title
  createdAt TEXT NOT NULL,         -- ISO 8601
  updatedAt TEXT NOT NULL,
  meta TEXT                        -- JSON metadata {tags, status, ...}
);

CREATE INDEX idx_threads_module ON threads(module);
CREATE INDEX idx_threads_updated ON threads(updatedAt DESC);

-- Messages (conversation turns)
CREATE TABLE messages (
  id TEXT PRIMARY KEY,             -- UUID
  threadId TEXT NOT NULL,          -- Foreign key to threads
  seq INTEGER NOT NULL,            -- Sequence number (1, 2, 3, ...)
  role TEXT NOT NULL,              -- 'user' | 'assistant' | 'system'
  content TEXT NOT NULL,           -- Message text
  createdAt TEXT NOT NULL,
  FOREIGN KEY (threadId) REFERENCES threads(id) ON DELETE CASCADE
);

CREATE INDEX idx_messages_thread ON messages(threadId, seq);
CREATE INDEX idx_messages_role ON messages(role);

-- Full-text search index
CREATE VIRTUAL TABLE messages_fts USING fts5(
  threadId,
  role,
  content,
  content=messages,
  content_rowid=rowid
);

-- Triggers to keep FTS in sync
CREATE TRIGGER messages_fts_insert AFTER INSERT ON messages
BEGIN
  INSERT INTO messages_fts(rowid, threadId, role, content)
  VALUES (new.rowid, new.threadId, new.role, new.content);
END;

CREATE TRIGGER messages_fts_delete AFTER DELETE ON messages
BEGIN
  DELETE FROM messages_fts WHERE rowid = old.rowid;
END;
```

### API Design (MCP Tools)

**Thread management:**
```typescript
// Create thread
odei.history.threads.create.v1({
  module: 'discuss',
  title?: 'Constitutional alignment review',
  meta?: { tags: ['values', 'goals'] }
});
// Returns: { id: 'thread-uuid', createdAt: '...' }

// List threads
odei.history.threads.list.v1({
  module?: 'discuss',
  limit?: 10,
  offset?: 0
});
// Returns: [{ id, module, title, createdAt, messageCount }, ...]

// Get thread with messages
odei.history.threads.get.v1({
  threadId: 'thread-uuid',
  limit?: 25,      // Last N messages
  beforeSeq?: 50   // Pagination
});
// Returns: { thread: {...}, messages: [...] }

// Search threads
odei.history.threads.search.v1({
  q: 'autonomy goals',
  module?: 'discuss',
  limit?: 10
});
// Returns: [{ thread: {...}, matchedMessages: [...] }]
```

**Message management:**
```typescript
// Append message
odei.history.threads.appendMessage.v1({
  threadId: 'thread-uuid',
  role: 'user',
  body: 'What values relate to autonomy?'
});
// Returns: { id: 'message-uuid', seq: 42 }

// Recent messages (across threads)
odei.history.messages.recent.v1({
  module?: 'discuss',
  since?: '2025-12-20T00:00:00Z',
  limit?: 50
});
// Returns: [{ thread: {...}, message: {...} }, ...]
```

### Session Continuity Protocol

**At session start, agents MUST:**

1. **Fetch latest thread:**
```typescript
const threads = await odei.history.threads.list.v1({
  module: 'discuss',
  limit: 1
});
const latestThread = threads[0];
```

2. **Load recent messages:**
```typescript
const { messages } = await odei.history.threads.get.v1({
  threadId: latestThread.id,
  limit: 25  // Last 25 turns
});
```

3. **Synthesize continuity snapshot:**
```markdown
Last session: 2025-12-24 20:30 (Doha)
Topic: Q1 goal planning
Mood: Focused, slightly fatigued
Outstanding: Review Tipz initiative risks
Promises: Send updated OKRs by Friday
```

4. **Reference in first reply:**
```
Picking up from where we paused yesterday evening.
You were planning Q1 goals, specifically the Tipz initiative.
Still want to review those risks, or shift focus?
```

### Search Implementation

**Full-text search example:**
```typescript
// User asks: "What did we discuss about autonomy last week?"
const results = await odei.history.threads.search.v1({
  q: 'autonomy',
  module: 'discuss',
  limit: 10
});

// Results include:
// - Thread: "Values review - Dec 18"
// - Matched messages: "Value: Autonomy as core principle..."
// - Timestamp: 2025-12-18T14:22:00Z
```

**FTS5 query syntax:**
- Simple: `"autonomy"` (exact word)
- Phrase: `"financial freedom"` (exact phrase)
- Prefix: `"goal*"` (goal, goals, goaling)
- Boolean: `"autonomy AND goals"` (both required)
- Proximity: `"autonomy NEAR/5 freedom"` (within 5 words)

### Backup & Export

**Export format (JSON Lines):**
```json
{"thread":{"id":"...","module":"discuss","title":"...","createdAt":"..."}}
{"message":{"id":"...","threadId":"...","seq":1,"role":"user","content":"..."}}
{"message":{"id":"...","threadId":"...","seq":2,"role":"assistant","content":"..."}}
```

**Tools:**
```bash
# Backup database
cp data/history.sqlite backups/history-2025-12-25.sqlite

# Export to JSON
sqlite3 data/history.sqlite <<EOF
  SELECT json_object(
    'thread', json_object('id', id, 'module', module, ...)
  ) FROM threads;
EOF
```

### Performance Characteristics

**Measured on MacBook Pro M1 Max:**
- Insert message: <1ms
- Query last 25 messages: 3ms
- Full-text search (1000 messages): 8ms
- List threads (100 threads): 2ms
- Database size: ~1KB per message (1000 messages ≈ 1MB)

**Optimizations:**
- `better-sqlite3` uses synchronous API (no callback overhead)
- FTS5 maintains separate index (parallel updates)
- WAL mode for concurrent reads during writes

## Consequences

### Positive

1. **✅ Session continuity:** Claude remembers last conversation context
2. **✅ Searchable history:** Find past discussions in <10ms
3. **✅ No external dependencies:** SQLite embedded, no service to manage
4. **✅ Privacy:** All data stored locally (no cloud sync)
5. **✅ Reliability:** ACID transactions prevent data loss
6. **✅ Portability:** Single file, easy to backup/transfer
7. **✅ Performance:** 1000+ messages queried in <10ms

### Negative

1. **⚠️ Single file:** Database corruption affects all history
2. **⚠️ No real-time sync:** Changes not visible across multiple app instances
3. **⚠️ Search limitations:** FTS5 lacks semantic understanding (keyword-only)
4. **⚠️ Storage growth:** 1GB database after ~1M messages (mitigated by archival)
5. **⚠️ No encryption:** Sensitive conversations stored in plain text

### Operational Impact

**Database location:**
```
/Users/ai/ODEI/data/history.sqlite
```

**Backup strategy:**
```bash
# Daily backup (automated)
cp data/history.sqlite backups/history-$(date +%Y%m%d).sqlite

# Keep last 30 days
find backups/ -name "history-*.sqlite" -mtime +30 -delete
```

**Monitoring:**
```sql
-- Database size
SELECT page_count * page_size / 1024 / 1024 AS size_mb
FROM pragma_page_count(), pragma_page_size();

-- Message count by module
SELECT module, COUNT(*) FROM threads
JOIN messages ON threads.id = messages.threadId
GROUP BY module;
```

**Maintenance:**
```sql
-- Vacuum (compact database)
VACUUM;

-- Analyze (optimize query planner)
ANALYZE;

-- Archive old threads (>6 months)
DELETE FROM threads
WHERE updatedAt < datetime('now', '-6 months');
```

## Alternatives Considered

### 1. Neo4j (Graph Database)

**Approach:** Store conversations as nodes in constitutional graph

**Pros:**
- Unified storage (all data in Neo4j)
- Graph relationships (link conversations to Goals)
- Cypher queries

**Cons:**
- Overkill for linear conversation threads
- Slower than SQLite for sequential reads
- Mixes conversation data with constitutional graph

**Rejected:** Conversations are inherently sequential, not graph-structured.

### 2. JSON Files

**Approach:** One JSON file per thread

**Pros:**
- Simple implementation
- Human-readable
- No database setup

**Cons:**
- No full-text search
- Poor concurrent access
- Slow for large files (must parse entire file)
- No transactions (partial writes during crashes)

**Rejected:** Performance and reliability requirements rule this out.

### 3. PostgreSQL

**Approach:** Use Postgres with JSONB columns

**Pros:**
- Powerful query capabilities
- Horizontal scaling
- Advanced full-text search

**Cons:**
- Requires separate service (Postgres server)
- Overkill for single-user app
- Slower than SQLite for local reads
- More complex deployment

**Rejected:** SQLite sufficient for local-first app.

### 4. LevelDB / RocksDB

**Approach:** Embedded key-value store

**Pros:**
- Fast writes
- Compact storage
- No SQL parsing overhead

**Cons:**
- No native full-text search
- Requires building indexes manually
- Less mature Node.js bindings

**Rejected:** SQL's expressiveness worth the overhead.

### 5. In-Memory Only

**Approach:** Keep conversation history in RAM

**Pros:**
- Maximum speed
- Simple implementation

**Cons:**
- Lost on app restart
- RAM limits history size
- No persistence

**Rejected:** Session continuity requires persistent storage.

## References

- SQLite Documentation: https://www.sqlite.org/docs.html
- FTS5 Full-Text Search: https://www.sqlite.org/fts5.html
- better-sqlite3: https://github.com/WiseLibs/better-sqlite3
- ODEI History Server: `/Users/ai/ODEI/servers/odei-history/`

## Revision History

- **2025-12-25:** Initial ADR documenting SQLite adoption for conversation history
