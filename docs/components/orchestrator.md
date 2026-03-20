# Orchestrator

Single long-running process. Entry point of the system.

## Modules

```
orchestrator
├── telegram_bot        # Telegram Bot API polling, message routing
├── session_manager     # Session CRUD, maps chat/topic to container
├── container_manager   # Docker spawn, kill, prune
├── ipc_watcher         # Polls output/ dirs for agent responses
├── credential_proxy    # HTTP reverse proxy, injects API key
└── db                  # SQLite access layer
```

## Startup sequence

```
1. Init DB (run migrations if needed)
2. Kill orphaned containers from previous run
3. Reset orphaned sessions to "stopped" in DB
4. Start credential proxy (port 3001)
5. Start IPC watcher loop (1s interval)
6. Start prune loop (5min interval)
7. Connect Telegram bot (start polling)
```

## Shutdown (SIGTERM / SIGINT)

```
1. Stop Telegram polling
2. Write _close to all running agents' input/ dirs
3. Wait up to 10s for containers to exit
4. docker kill any remaining containers
5. Stop credential proxy
6. Close DB
7. Exit
```

## Module responsibilities

### telegram_bot

- Long-polls Telegram Bot API for updates
- Checks `allowed_users` and `allowed_chats` tables, drops unauthorized
- Routes text messages to `session_manager`
- Handles `/kill` and `/status` commands
- Sends agent responses back to the correct chat/topic

### session_manager

- Maps `(chat_id, topic_id)` to a session
- If no active session: creates one, calls `container_manager.spawn()`
- If active session: pipes message via IPC `input/*.json`
- Updates session state in DB on lifecycle events

### container_manager

- `spawn(session_id)` — `docker run` with volume mounts per agent-runner spec, env vars per ADR-006, `.env` shadowed with `/dev/null`
- `kill(session_id)` — write `_close`, wait 10s, then `docker kill`
- `prune()` — every 5 min, remove containers in `stopped` state older than 30 min
- On startup: `docker ps --filter name=clawnim-*` to find orphans, kill them

Container naming: `clawnim-{session_id}`

### ipc_watcher

- Polls `data/ipc/*/output/` directories every 1s
- Reads `.json` files, parses agent messages
- Routes `type: "message"` to `telegram_bot` for delivery
- Routes `type: "done"` to `session_manager` to transition session to `stopped`
- Deletes processed files
- Moves unparseable files to `data/ipc/errors/`

### credential_proxy

- HTTP server on port 3001
- Reads `ANTHROPIC_API_KEY` from `.env` at startup
- Strips incoming `x-api-key`, injects real key
- Forwards to `https://api.anthropic.com`
- Returns 502 on upstream failure

### db

- SQLite with WAL mode
- Schema per ADR-004
- Runs migrations on startup
- Provides typed query functions, no ORM

## Logging

All output to stdout/stderr. No structured format. Errors to stderr, everything else to stdout.
