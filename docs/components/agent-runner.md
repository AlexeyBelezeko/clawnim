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

Nanoclaw pipes input via stdin (`echo JSON | docker run -i`). This does not work with `docker run -d` (detached mode) — stdin EOF arrives before the container reads it.

**Change:** Orchestrator writes input JSON to a file and mounts it:
```
-v {session_dir}/input.json:/tmp/input.json:ro
```

Entrypoint reads from the mounted file instead of stdin:
```bash
# nanoclaw original:
cat > /tmp/input.json
node /tmp/dist/index.js < /tmp/input.json

# clawnim change:
node /tmp/dist/index.js < /tmp/input.json
# (remove the `cat` line — file is already mounted)
```

### 2. SDK session resume

Nanoclaw passes its own group ID as the `resume` session ID. The Claude Agent SDK requires a UUID for `--resume`.

**Change:** Agent-runner must capture the SDK's own `session_id` (UUID) from the first query's result messages and use that for subsequent `resume` calls. Do not pass the clawnim session ID (e.g. `telegram_12345`) to `resume`.

### 3. `.claude` directory structure

The Claude Agent SDK writes debug logs to `/home/node/.claude/debug/`. If the directory doesn't exist, the SDK crashes.

**Change:** Orchestrator must create these subdirectories before first spawn:
```
data/sessions/{session_id}/.claude/debug/
data/sessions/{session_id}/.claude/projects/
```

### 4. Agent-runner source seeding

Nanoclaw mounts per-group source at `/app/src/`. On first run this directory is empty, which shadows the built-in source files causing TypeScript compilation to fail.

**Change:** Orchestrator copies default source files from `container/agent-runner/src/` into the session's agent-runner-src directory before first spawn, if empty.

### 5. Response streaming

Nanoclaw's agent-runner uses the `send_message` MCP tool for real-time messages to the user. Additionally, the agent-runner should forward SDK `assistant` messages (text content blocks) via IPC `output/*.json` with `type: "message"` so the user sees the agent's responses without relying solely on the MCP tool.

## Volume mounts

| Container path | Host path | Access |
|---|---|---|
| `/workspace/ipc/` | `data/ipc/{session_id}/` | rw |
| `/workspace/project/` | project repo root (main chat only) | ro |
| `/workspace/project/.env` | `/dev/null` | ro |
| `/workspace/group/` | `data/sessions/{session_id}/group/` | rw |
| `/workspace/global/CLAUDE.md` | repo `CLAUDE.md` | ro |
| `/workspace/global/skills/` | repo skills directory | ro |
| `/home/node/.claude/` | `data/sessions/{session_id}/.claude/` | rw |
| `/app/src/` | `data/sessions/{session_id}/agent-runner-src/` | rw |
| `/tmp/input.json` | `data/sessions/{session_id}/input.json` | ro |

## Container environment

```
ANTHROPIC_BASE_URL=http://host.docker.internal:{proxy_port}
ANTHROPIC_API_KEY=placeholder
```
