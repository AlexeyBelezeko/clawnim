# ADR-002: File-Based IPC Protocol

**Status:** Accepted
**Date:** 2026-03-20

## Decision

Orchestrator and agent containers communicate via JSON files on a shared Docker volume.
Protocol reuses nanoclaw's approach adapted for single-user Telegram setup.

### Directory structure

Host side:
```
data/ipc/{session_id}/
  input/               # orchestrator -> agent
  output/              # agent -> orchestrator
```

Container sees:
```
/workspace/ipc/
  input/               # read
  output/              # write
```

### File naming

```
{timestamp_ms}-{random_6char}.json
```

Example: `1711929600000-a3f8k2.json`

### Atomic writes

Write to `.tmp`, then `rename`. Prevents reading partial files.

```
write(path + ".tmp", data)
rename(path + ".tmp", path)
```

### Message formats

**Task assignment** (`input/*.json`, orchestrator writes):
```json
{
  "type": "task",
  "prompt": "Fix the failing test in auth module",
  "context": {}
}
```

**Follow-up message** (`input/*.json`, orchestrator writes):
```json
{
  "type": "message",
  "text": "Also check the login endpoint"
}
```

**Shutdown signal** (`input/_close`, empty file, orchestrator writes):
Agent must exit gracefully upon detecting this file.

**Agent response** (`output/*.json`, agent writes):
```json
{
  "type": "message",
  "text": "Fixed the test. Root cause was...",
  "status": "in_progress"
}
```

**Agent completion** (`output/*.json`, agent writes):
```json
{
  "type": "done",
  "text": "All changes committed. Summary: ...",
  "status": "completed"
}
```

### Polling

| Component | Directory | Interval |
|---|---|---|
| Orchestrator | `output/` | 1000ms |
| Agent | `input/` | 1000ms |

### File cleanup

Orchestrator deletes processed files from `output/` after reading.
Agent deletes processed files from `input/` after reading.
Failed files moved to `data/ipc/errors/{session_id}-{filename}` for debugging.

### Volume mounts per container

| Container path | Host path | Access |
|---|---|---|
| `/workspace/ipc` | `data/ipc/{session_id}` | rw |
| `/workspace/project` | project repo clone | rw |
