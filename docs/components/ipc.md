# IPC Layer

File-based communication between orchestrator and agent containers via shared Docker volumes.

## Directory structure

Host side:
```
data/ipc/{session_id}/
  input/               # orchestrator -> agent
  output/              # agent -> orchestrator
```

Container sees:
```
/workspace/ipc/
  input/
  output/
```

## File naming

```
{timestamp_ms}-{random_6char}.json
```

Files are processed in alphabetical order (timestamp prefix ensures chronological ordering).

## Atomic writes

All JSON files written atomically: write to `.tmp`, then `rename`. Prevents reading partial files.

## Message formats

### Orchestrator -> Agent (`input/`)

**Initial input** — mounted as a file, not via stdin (detached containers cannot receive piped stdin):
```json
{
  "prompt": "Fix the failing test in auth module",
  "sessionId": "telegram_12345"
}
```

**Task assignment** (`input/*.json`):
```json
{
  "type": "task",
  "prompt": "Fix the failing test in auth module",
  "context": {}
}
```

**Follow-up message** (`input/*.json`):
```json
{
  "type": "message",
  "text": "Also check the login endpoint"
}
```

**Shutdown signal** (`input/_close`, empty file):
Agent must exit gracefully upon detecting this file.

### Agent -> Orchestrator (`output/`)

**Agent response** (in-progress update, forwarded to user):
```json
{
  "type": "message",
  "text": "Fixed the test. Root cause was...",
  "status": "in_progress"
}
```

**Query completion** (signals query finished, NOT forwarded to user):
```json
{
  "type": "done",
  "text": "...",
  "status": "completed"
}
```

## Semantics

- `type: "message"` — assistant response, forwarded to user immediately.
- `type: "done"` — current query finished. The orchestrator must NOT forward `done` text to the user — the response was already delivered via `type: "message"`. The orchestrator must NOT transition the session to `stopped`. The agent stays alive, polling `input/` for follow-up messages.
- Session transitions to `stopped` only when the container process actually exits. Orchestrator detects this by periodically checking container status.

## Polling

| Component | Directory | Interval |
|---|---|---|
| Orchestrator | `output/` | 1000ms |
| Agent | `input/` | 500ms |

## File cleanup

- Orchestrator deletes processed files from `output/` after reading.
- Agent deletes processed files from `input/` after reading.
- Failed/unparseable files moved to `data/ipc/errors/{session_id}-{filename}`.

## Dead container detection

Orchestrator periodically checks (every ~10s) if running sessions still have a live container. If the container has exited:
1. Remove the container so the name can be reused.
2. Transition session to `stopped`.
