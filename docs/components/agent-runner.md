# Agent Runner

Reuses nanoclaw's agent-runner as-is. Source: `container/agent-runner/` in [nanoclaw](https://github.com/qwibitai/nanoclaw).

## What we take from nanoclaw

Copy these files directly:

| Nanoclaw path | Purpose |
|---|---|
| `container/agent-runner/src/index.ts` | Agent entry point, query loop, follow-up polling |
| `container/agent-runner/src/ipc-mcp-stdio.ts` | MCP server exposing `send_message` tool to Claude Code |
| `container/Dockerfile` | Container image (node:22-slim + chromium + claude-code + SDK) |
| `container/agent-runner/package.json` | Dependencies |
| `container/agent-runner/tsconfig.json` | TypeScript config |

The entrypoint script, Dockerfile, IPC MCP server, and query loop logic should work without modification.

## Changes needed for clawnim

### 1. Input delivery

Nanoclaw pipes input via stdin. This does not work with detached containers — stdin EOF arrives before the container reads it.

**Change:** Orchestrator mounts input JSON as a file. Entrypoint reads from the mounted file instead of stdin (remove the `cat > /tmp/input.json` line).

### 2. SDK session resume

Nanoclaw passes its own group ID as the `resume` session ID. The Claude Agent SDK requires a UUID for resume.

**Change:** Agent-runner must capture the SDK's own `session_id` (UUID) from the first query's result messages and use that for subsequent resume calls. Do not pass the clawnim session ID (e.g. `telegram_12345`) to resume.

### 3. `.claude` directory structure

The Claude Agent SDK writes debug logs to `/home/node/.claude/debug/`. If the directory doesn't exist, the SDK crashes.

**Change:** Orchestrator must create `debug/` and `projects/` subdirectories inside the `.claude` mount before first spawn.

### 4. Agent-runner source seeding

Nanoclaw mounts per-group source at `/app/src/`. On first run this directory is empty, which shadows the built-in source files causing compilation to fail.

**Change:** Orchestrator copies default agent-runner source into the session directory before first spawn, if empty.

### 5. Response streaming

Nanoclaw's agent-runner uses the `send_message` MCP tool for real-time messages to the user. Additionally, the agent-runner should forward SDK assistant messages (text content blocks) via IPC output with `type: "message"` so the user sees the agent's responses without relying solely on the MCP tool.

## Volume mounts and container environment

See `docs/components/orchestrator.md` (container_manager) and `docs/components/credential-proxy.md`.
