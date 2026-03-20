# ADR-006: Credential Proxy

**Status:** Accepted
**Date:** 2026-03-20

## Decision

Orchestrator runs an HTTP reverse proxy that injects Anthropic API credentials into agent requests. Agents never hold real API keys.

### Proxy

- HTTP server inside the orchestrator process, listening on port `3001`
- Forwards requests to `https://api.anthropic.com` (or `ANTHROPIC_BASE_URL` from `.env`)
- On each request: strips incoming `x-api-key` header, injects real key from `.env`
- Returns `502` on upstream errors

### Bind address

| Platform | Bind address |
|---|---|
| macOS (Docker Desktop) | `127.0.0.1` |
| Linux | `docker0` bridge IP (e.g. `172.17.0.1`), fallback `0.0.0.0` |

On Linux, containers need `--add-host=host.docker.internal:host-gateway`.

### Container environment

```
ANTHROPIC_BASE_URL=http://host.docker.internal:3001
ANTHROPIC_API_KEY=placeholder
```

Agent SDK sends requests to proxy with placeholder key. Proxy swaps it for the real key before forwarding.

### Secret isolation

- Real `ANTHROPIC_API_KEY` read from `.env` at orchestrator startup
- `.env` file shadowed inside containers by mounting `/dev/null` over it:
  ```
  -v /dev/null:/workspace/project/.env:ro
  ```
- Agent containers have no access to real credentials
