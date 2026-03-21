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

The agent-runner forwards SDK assistant messages (text content blocks) via IPC output with `type: "message"`. On query completion, it writes `type: "done"` as a signal only — the orchestrator does NOT forward `done` text to the user since the response was already sent via `type: "message"`.

### 7. Idle timeout

Nanoclaw relies on the orchestrator's group-queue idle timeout (30 min). This is fragile — if the orchestrator misses a dead container, the agent loops forever.

**Change:** Agent-runner itself exits after 30 minutes of no input. The timer resets each time `input/` delivers a message. On timeout, the process writes a final `type: "done"` with `status: "idle_timeout"` to `output/` and exits cleanly. The orchestrator detects the exit via its dead-container check and transitions the session to `stopped`.

### 6. Placeholder API key format

The container receives `ANTHROPIC_API_KEY` with a placeholder value (routed through the credential proxy). The placeholder must look like a real API key (e.g. `sk-ant-api03-placeholder...`) to pass the Claude CLI's local format validation. A literal `placeholder` string will be rejected on session resume.

## Volume mounts and container environment

See `docs/components/orchestrator.md` (container_manager) and `docs/components/credential-proxy.md`.
