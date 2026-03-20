# Nanoclaw Reference

AI agent orchestration platform. Primary inspiration for clawnim.

Repository: https://github.com/qwibitai/nanoclaw

## Key source files

### Orchestrator (host side)

| File | Purpose |
|---|---|
| `src/index.ts` | Entry point, starts all subsystems |
| `src/ipc.ts` | IPC watcher, polls agent output dirs |
| `src/group-queue.ts` | Per-group container queue, follow-up piping, idle timeout |
| `src/container-runner.ts` | Builds Docker args, volume mounts, env vars, runs containers |
| `src/container-runtime.ts` | Docker runtime abstraction, host gateway, bind address |
| `src/credential-proxy.ts` | HTTP reverse proxy, injects API key into agent requests |
| `src/db.ts` | SQLite schema, migrations, query functions |
| `src/task-scheduler.ts` | Cron/interval task scheduling loop |
| `src/config.ts` | Constants: ports, timeouts, concurrency limits |
| `src/env.ts` | Reads `.env` without polluting `process.env` |
| `src/mount-security.ts` | Blocks `.env`, `.ssh`, credentials from container mounts |
| `src/types.ts` | TypeScript interfaces for all data structures |

### Agent runner (container side)

| File | Purpose |
|---|---|
| `container/agent-runner/src/index.ts` | Agent entry point, query loop, follow-up polling |
| `container/agent-runner/src/ipc-mcp-stdio.ts` | MCP server exposing IPC tools to Claude Code |
| `container/Dockerfile` | Container image definition |
| `container/build.sh` | Image build script |

### Other

| File | Purpose |
|---|---|
| `CLAUDE.md` | Project instructions for Claude Code |
| `docs/nanoclaw-architecture-final.md` | Architecture overview |

## Architecture summary

- Single Node.js process orchestrator
- WhatsApp + Telegram + Discord channels (multi-channel, unlike clawnim)
- Docker containers for agents, one per group
- File-based IPC: `messages/`, `tasks/`, `input/` dirs per group
- SQLite (`store/messages.db`) for all persistent state
- Credential proxy on port 3001
- Claude Agent SDK (`query()`) inside containers, not CLI
- MCP server for IPC tools (`send_message`, `schedule_task`, etc.)
- Per-group agent-runner source customization via bind mounts

## Key constants

| Constant | Value | File |
|---|---|---|
| `CREDENTIAL_PROXY_PORT` | 3001 | `src/config.ts` |
| `IPC_POLL_INTERVAL` | 1000ms | `src/ipc.ts` |
| `IPC_POLL_MS` (container) | 500ms | `container/agent-runner/src/index.ts` |
| `IDLE_TIMEOUT` | 30 min | `src/group-queue.ts` |
| `MAX_CONCURRENT_CONTAINERS` | 5 | `src/config.ts` |
| `SCHEDULER_POLL_INTERVAL` | 60s | `src/task-scheduler.ts` |
