# Orchestrator

Single long-running process. Entry point of the system.

## Modules

```
orchestrator
├── channel             # Channel interface + implementations (telegram, ...)
├── session_manager     # Session CRUD, maps chat/topic to container
├── container_manager   # Docker spawn, kill, prune
├── ipc_watcher         # Polls output/ dirs for agent responses
├── credential_proxy    # HTTP reverse proxy, injects API key
└── db                  # SQLite access layer
```

## Configuration

`.env` file:
```
TELEGRAM_BOT_TOKEN=...
ANTHROPIC_API_KEY=...
MAIN_CHAT_ID=...          # auto-seeds allowed_users + allowed_chats on startup
DATA_DIR=./data            # default
CREDENTIAL_PROXY_PORT=3001 # default
ANTHROPIC_BASE_URL=https://api.anthropic.com  # default
```

## Startup sequence

```
1. Init DB (run migrations if needed)
2. Seed allowed_users/allowed_chats from MAIN_CHAT_ID if set
3. Kill orphaned containers from previous run
4. Reset orphaned sessions to "stopped" in DB
5. Start credential proxy
6. Start IPC watcher loop (1s interval)
7. Start prune loop (5min interval)
8. Start channels (Telegram, ...)
```

## Shutdown (SIGTERM / SIGINT)

```
1. Stop all channels
2. Write _close to all running agents' input/ dirs
3. Wait up to 10s for containers to exit
4. docker kill any remaining containers
5. Stop credential proxy
6. Close DB
7. Exit
```

## Module responsibilities

### channel

Interface for user-facing messaging. Each channel implementation provides:
- `start()` — connect and begin receiving messages
- `stop()` — disconnect gracefully
- `send(chat_id, text)` — deliver message to a chat
- `on_message(callback)` — incoming message handler

Incoming messages normalized to: `{ channel, chat_id, topic_id, sender_id, sender_name, text }`

Channels handle their own access control (check `allowed_users`, `allowed_chats`), command parsing, and message splitting.

See `docs/components/telegram.md` for the Telegram implementation.

### session_manager

- Maps `(chat_id, topic_id)` to a session
- If no active session: creates one, calls `container_manager.spawn()`
- If active session: pipes message via IPC `input/*.json`
- Updates session state in DB on lifecycle events

### container_manager

- `spawn(session_id)` — start detached container with volume mounts per agent-runner spec, credential proxy env vars, `.env` shadowed with `/dev/null`. Input JSON mounted as file (not piped via stdin — detached containers cannot receive stdin).
- `kill(session_id)` — write `_close`, wait 10s, then force kill
- `remove(session_id)` — remove stopped container, frees the name for reuse
- `prune()` — every 5 min, remove containers in `stopped` state older than 30 min
- On startup: find orphaned containers by name prefix, kill them

Container naming: `clawnim-{session_id}`

Per-session directory setup before first spawn:
- `data/sessions/{session_id}/.claude/debug/` — SDK debug logs (must exist or SDK crashes)
- `data/sessions/{session_id}/.claude/projects/` — SDK project data
- `data/sessions/{session_id}/group/` — working directory
- `data/sessions/{session_id}/agent-runner-src/` — seeded from default agent-runner source if empty

### ipc_watcher

- Polls `data/ipc/*/output/` directories every 1s
- Reads `.json` files, parses agent messages
- Routes `type: "message"` to the originating channel for delivery
- Routes `type: "done"` — forwards text to channel. Does NOT stop the session (agent stays alive for follow-ups)
- Deletes processed files
- Moves unparseable files to `data/ipc/errors/`
- Every ~10s, checks for dead containers: if a running session's container has exited, removes it and transitions session to `stopped`

See `docs/components/ipc.md` for protocol details.

### credential_proxy

See `docs/components/credential-proxy.md`.

### db

See `docs/components/db.md`.

## Logging

All output to stdout/stderr. No structured format. Errors to stderr, everything else to stdout.
