# Credential Proxy

HTTP reverse proxy inside the orchestrator process. Injects Anthropic API credentials into agent requests. Agents never hold real API keys.

## Behavior

- Listens on configurable port (default `3001`)
- On each request: strips incoming `x-api-key` header, injects real key from `.env`
- Forwards to upstream (`ANTHROPIC_BASE_URL`, default `https://api.anthropic.com`)
- Returns `502` on upstream errors
- No streaming — buffers full request/response

## Bind address

| Platform | Bind address |
|---|---|
| macOS (Docker Desktop) | `127.0.0.1` |
| Linux | `docker0` bridge IP (e.g. `172.17.0.1`), fallback `0.0.0.0` |

On Linux, containers need `--add-host=host.docker.internal:host-gateway`.

## Container environment

Containers are started with:
```
ANTHROPIC_BASE_URL=http://host.docker.internal:{proxy_port}
ANTHROPIC_API_KEY=placeholder
```

Agent SDK sends requests to proxy with placeholder key. Proxy swaps it for the real key before forwarding.

## Secret isolation

- Real `ANTHROPIC_API_KEY` read from `.env` at orchestrator startup
- `.env` file shadowed inside containers by mounting `/dev/null` over it
- Agent containers have no access to real credentials
