# Credential Proxy

HTTP reverse proxy inside the orchestrator process. Injects Anthropic API credentials into agent requests. Agents never hold real API keys.

## Behavior

- Listens on configurable port (default `3001`)
- On each request: strips incoming `x-api-key` header, injects real key from `.env`
- Forwards to upstream (`ANTHROPIC_BASE_URL`, default `https://api.anthropic.com`)
- Returns `502` on upstream errors
- No streaming — buffers full request/response

## Token types

Both standard API keys (`sk-ant-api03-*`) and OAuth tokens (`sk-ant-oat01-*`) work as `x-api-key`. The proxy always uses API key mode — inject as `x-api-key` regardless of token type.

`.env` accepts either variable:
```
ANTHROPIC_API_KEY=...
# or
CLAUDE_CODE_OAUTH_TOKEN=...
```

Both are injected as `x-api-key` through the proxy.

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
ANTHROPIC_API_KEY=sk-ant-api03-placeholder...
```

The placeholder key must look like a real API key (start with `sk-ant-api03-`) to pass the Claude CLI's local format validation. A literal `placeholder` string will be rejected.

Agent SDK sends requests to proxy with placeholder key. Proxy strips it and injects the real key before forwarding.

## Secret isolation

- Real key read from `.env` at orchestrator startup
- `.env` file shadowed inside containers by mounting `/dev/null` over it
- Agent containers have no access to real credentials

## Verification

After starting the proxy, verify end-to-end that the injected key reaches upstream:
```
curl http://localhost:3001/v1/messages \
  -H "x-api-key: fake" \
  -H "anthropic-version: 2023-06-01" \
  -d '{"model":"...","max_tokens":1,"messages":[{"role":"user","content":"hi"}]}'
```
Should return a valid API response (not `invalid x-api-key`).
