# ADR-004: SQLite Data Model

**Status:** Accepted
**Date:** 2026-03-20

## Decision

Single SQLite file at `data/store.db`. WAL mode enabled.

### Schema version

```sql
CREATE TABLE meta (
  key   TEXT PRIMARY KEY,
  value TEXT NOT NULL
);
-- initial row: ('schema_version', '1')
```

Migration scripts run on startup. Orchestrator compares `schema_version` to expected version and applies migrations sequentially.

### Sessions

```sql
CREATE TABLE sessions (
  id           TEXT PRIMARY KEY,  -- "{chat_id}" or "{chat_id}_{topic_id}"
  chat_id      INTEGER NOT NULL,
  topic_id     INTEGER,           -- NULL if no topics
  container_id TEXT,              -- Docker container ID, NULL if not running
  state        TEXT NOT NULL,     -- creating, running, stopping, stopped, pruned
  created_at   TEXT NOT NULL,     -- ISO 8601
  updated_at   TEXT NOT NULL
);
```

### Tasks

```sql
CREATE TABLE tasks (
  id         TEXT PRIMARY KEY,  -- "{timestamp_ms}-{random_6char}"
  session_id TEXT NOT NULL REFERENCES sessions(id),
  prompt     TEXT NOT NULL,
  status     TEXT NOT NULL,     -- queued, running, completed, failed, killed
  created_at TEXT NOT NULL,
  started_at TEXT,
  ended_at   TEXT
);
```

### Task run logs

```sql
CREATE TABLE task_run_logs (
  id          INTEGER PRIMARY KEY AUTOINCREMENT,
  task_id     TEXT NOT NULL REFERENCES tasks(id),
  run_at      TEXT NOT NULL,
  duration_ms INTEGER NOT NULL,
  status      TEXT NOT NULL,     -- success, error
  result      TEXT,
  error       TEXT
);
```

### Messages

All inbound (Telegram -> orchestrator) and outbound (orchestrator -> Telegram) messages stored.

```sql
CREATE TABLE messages (
  id             INTEGER PRIMARY KEY AUTOINCREMENT,
  session_id     TEXT NOT NULL REFERENCES sessions(id),
  sender         TEXT NOT NULL,     -- telegram user ID
  sender_name    TEXT,
  content        TEXT NOT NULL,
  is_from_me     INTEGER DEFAULT 0,
  is_bot_message INTEGER DEFAULT 0,
  created_at     TEXT NOT NULL
);
```

### Router state

Generic key-value store for orchestrator runtime state.

```sql
CREATE TABLE router_state (
  key   TEXT PRIMARY KEY,
  value TEXT NOT NULL
);
```

### Access control

```sql
CREATE TABLE allowed_users (
  telegram_user_id INTEGER PRIMARY KEY,
  username         TEXT,
  added_at         TEXT NOT NULL
);

CREATE TABLE allowed_chats (
  telegram_chat_id INTEGER PRIMARY KEY,
  title            TEXT,
  added_at         TEXT NOT NULL
);
```

Unauthorized messages silently dropped.

### Indexes

```sql
CREATE INDEX idx_messages_created_at ON messages(created_at);
CREATE INDEX idx_task_run_logs_task_id ON task_run_logs(task_id, run_at);
```

### Configuration

Secrets (API keys, bot token) in `.env` file, never in SQLite. SQLite stores only runtime state: allowed users, allowed chats, router state.
