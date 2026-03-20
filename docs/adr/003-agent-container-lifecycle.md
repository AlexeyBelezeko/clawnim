# ADR-003: Agent Container Lifecycle

**Status:** Accepted
**Date:** 2026-03-20

## Decision

### Container image

Reuse nanoclaw's agent-runner image (Node.js + Claude Code + tools). Custom images are a future concern.

### Concurrency model

- Multiple containers run concurrently
- One container per Telegram chat (or per topic in group chats)
- If a task arrives for a chat that already has a running container, the message is piped via `input/*.json` (follow-up, not a new container)
- No global concurrency limit initially

### Session persistence

Sessions are persistent. Agent session data is mounted between runs:

| Container path | Host path |
|---|---|
| `/home/node/.claude` | `data/sessions/{session_id}/.claude` |

Session ID maps 1:1 to chat/topic.

### Project code access

- **Main chat** container gets project repo mounted **read-only** at `/workspace/project`
- **Other chat/topic** containers get no source code mount
- Agents that need code access must clone from git remote themselves

### Container states

```
creating -> running -> stopping -> stopped -> pruned
```

- `creating` — Docker container being spawned, volumes mounted
- `running` — agent processing, IPC active
- `stopping` — `_close` written to `input/`, waiting for agent exit
- `stopped` — container exited, kept for debugging
- `pruned` — container and IPC directory removed

### Spawn

1. Orchestrator creates `data/ipc/{session_id}/input/` and `output/`
2. Docker container started with volume mounts per ADR-002
3. Initial task written to `input/{timestamp}-{random}.json`
4. Container state set to `running` in SQLite

### Shutdown

Two paths:

**Graceful (agent completes):**
1. Agent writes `type: "done"` to `output/`
2. Orchestrator reads it, sets state to `stopped`
3. Container exits on its own

**Forced (user kills via main chat):**
1. User sends kill command in Telegram
2. Orchestrator writes `_close` to `input/`
3. If container doesn't exit within 10s, `docker kill`
4. State set to `stopped`

### Cleanup

- Stopped containers kept for configurable duration (default: 30 min)
- Orchestrator runs prune loop, removes containers older than threshold
- IPC directory (`data/ipc/{session_id}/`) removed with container
- Session data (`data/sessions/{session_id}/`) preserved (persistent sessions)

### No health checks

No heartbeat or health check mechanism. Stuck containers are killed manually by the user via Telegram kill command.
