# Agent Runner

Runs inside the Docker container. Wraps Claude Code SDK, handles IPC with orchestrator.
Reuses nanoclaw's agent-runner approach.

## Container image

Base: `node:22-slim`

Installed:
- `chromium` + font/rendering deps
- `curl`, `git`
- `@anthropic-ai/claude-code` (global)
- `@anthropic-ai/claude-agent-sdk`
- `@modelcontextprotocol/sdk`

Runs as `node` user (non-root).

## Entry point

1. Read stdin — JSON blob with `{ prompt, sessionId }`
2. Clean up stale `_close` sentinel from previous run
3. Create `/workspace/ipc/input/` if missing
4. Drain any pending `input/*.json` files, append to prompt
5. Enter query loop

## Claude Code invocation

Uses Claude Agent SDK directly (`@anthropic-ai/claude-agent-sdk` `query()` function), not the CLI.

Key options:
- `cwd`: `/workspace/project` (if mounted) or `/workspace/group`
- `resume`: previous `sessionId` for session continuity
- `permissionMode`: `bypassPermissions`
- `allowedTools`: `Bash`, `Read`, `Write`, `Edit`, `Glob`, `Grep`, `WebSearch`, `WebFetch`, plus `mcp__clawnim__*`
- `mcpServers`: registers `clawnim` MCP server for IPC tools

## IPC — MCP server

Separate MCP server process (`ipc-mcp-stdio.ts`) registered with Claude Code SDK via `mcpServers` config. Exposes tools under `clawnim` namespace:

| Tool | Writes to | Purpose |
|---|---|---|
| `send_message` | `output/*.json` | Send message to user via Telegram |

All writes atomic (`.tmp` + `rename`).

## Follow-up messages

Orchestrator writes follow-up messages to `input/*.json`.

Agent-runner polls `/workspace/ipc/input/` every 500ms:
- **During active query**: pushes message into the SDK's `MessageStream`, arrives as new user turn in live conversation
- **Between queries**: triggers a new `query()` call with session resume

## Session lifecycle

Agent stays alive after completing a query:

```
start -> read stdin -> run query -> idle (poll input/) -> new message -> run query -> ...
                                                       -> _close      -> exit
```

No idle timeout inside the container. Orchestrator manages timeout externally.

## Shutdown (_close)

When `_close` file detected in `input/`:
- During query: ends the `MessageStream`, query finishes, loop exits
- Between queries: loop breaks immediately
- Process exits with code 0

## Container filesystem

| Path | Host path | Content | Access |
|---|---|---|---|
| `/workspace/ipc/input/` | `data/ipc/{session_id}/input/` | Messages from orchestrator | read |
| `/workspace/ipc/output/` | `data/ipc/{session_id}/output/` | Messages to orchestrator | write |
| `/workspace/project/` | project repo root (main chat only) | Project source code | read-only |
| `/workspace/project/.env` | `/dev/null` | Shadow `.env` to prevent secret leaks | read-only |
| `/workspace/group/` | `data/sessions/{session_id}/group/` | Per-session working directory | read-write |
| `/workspace/global/CLAUDE.md` | repo `CLAUDE.md` | Shared agent instructions | read-only |
| `/workspace/global/skills/` | repo skills directory | Shared skills/prompts | read-only |
| `/home/node/.claude/` | `data/sessions/{session_id}/.claude/` | Claude session persistence | read-write |
| `/app/src/` | `data/sessions/{session_id}/agent-runner-src/` | Agent-runner source | read-write |

### Secret isolation

`.env` in project root is shadowed by mounting `/dev/null` over it:
```
-v /dev/null:/workspace/project/.env:ro
```
Agent containers have no access to host secrets. API credentials are injected by the credential proxy (ADR-006).

## Entrypoint script

```bash
#!/bin/bash
set -e
cd /app && npx tsc --outDir /tmp/dist 2>&1 >&2
ln -s /app/node_modules /tmp/dist/node_modules
chmod -R a-w /tmp/dist
cat > /tmp/input.json
node /tmp/dist/index.js < /tmp/input.json
```

Recompiles TypeScript on every start. Per-session source is mounted at `/app/src/`, allowing customization per session.
