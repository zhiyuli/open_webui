# Open WebUI Local Setup

This repository runs Open WebUI with a local HTTP proxy (`privoxy`) that forwards traffic through a SOCKS5 upstream.

## What This Stack Does

- Runs Open WebUI in Docker (`openwebui` service)
- Runs Privoxy in Docker (`privoxy` service)
- Forces Open WebUI outbound HTTP(S) requests through Privoxy
- Uses selective Privoxy forwarding: default direct, with `nimbus.chpc.utah.edu` routed via SOCKS5 at `host.docker.internal:4567`

## Prerequisites

- Docker Desktop (or Docker Engine + Compose plugin)
- Git

## Quick Start

1. Open this repo:

```bash
cd /Users/u6047586/open_webui
```

2. Start services:

```bash
docker compose up -d
```

3. Check status:

```bash
docker compose ps
```

4. Open Open WebUI at:

- `http://localhost:3000`

## How Proxying Works In This Repo

Traffic path in this setup:

1. Open WebUI sends outbound requests using these env vars in `docker-compose.yml`:
   - `HTTP_PROXY=http://privoxy:8118`
   - `HTTPS_PROXY=http://privoxy:8118`
   - `ALL_PROXY=http://privoxy:8118`
2. The `privoxy` hostname resolves over the internal Docker network `webproxy-net`.
3. Privoxy listens on `0.0.0.0:8118` (inside its container).
4. Privoxy forwarding rules are:
   - `forward / .` (default: direct)
   - `forward-socks5t .nimbus.chpc.utah.edu host.docker.internal:4567 .` (override: route this host through SOCKS)
5. Rule precedence for `forward*` directives is last-match-wins, so specific overrides are placed after broad defaults.
6. `forward-socks5t` means DNS/hostname resolution is done through SOCKS (remote DNS), not locally.

## LLM Endpoint In This Setup

Open WebUI is configured with:

- `OPENAI_API_BASE_URL=http://nimbus.chpc.utah.edu:44142/v1`
- `ENABLE_OLLAMA_API=false`
- `AIOHTTP_CLIENT_TIMEOUT_MODEL_LIST=30`

So API calls from Open WebUI are intended to go:

- `openwebui` -> `privoxy` -> SOCKS5 at `host.docker.internal:4567` -> remote endpoint (`nimbus.chpc.utah.edu:44142`)

Notes:

- `ENABLE_OLLAMA_API=false` prevents repeated `host.docker.internal:11434` connection-refused errors when local Ollama is not part of this stack.
- `AIOHTTP_CLIENT_TIMEOUT_MODEL_LIST=30` increases model-list/verify timeout for slower proxy/upstream paths.

## NO_PROXY Behavior

These hosts bypass the proxy from inside the Open WebUI container:

- `localhost`
- `127.0.0.1`
- `::1`
- `host.docker.internal`
- `172.17.0.1`

This is controlled by:

- `NO_PROXY=localhost,127.0.0.1,::1,host.docker.internal,172.17.0.1`

## Current Runtime Notes

- `privoxy/config` currently uses selective forwarding:
  - `forward / .` for default direct routing
  - `forward-socks5t .nimbus.chpc.utah.edu host.docker.internal:4567 .` for remote LLM traffic
- Open WebUI currently disables Ollama API polling with `ENABLE_OLLAMA_API=false`.
- The model list timeout is set to `AIOHTTP_CLIENT_TIMEOUT_MODEL_LIST=30`.

## Files

- `docker-compose.yml`: service wiring, ports, env vars
- `privoxy/config`: proxy listener + SOCKS forwarding rules

## Stop Services

```bash
docker compose down
```

## Troubleshooting

- If `localhost:3000` is unreachable, verify both containers are `Up` with `docker compose ps`.
- If model calls fail, verify SOCKS is reachable at `host.docker.internal:4567` from Docker.
- If `3000` or `8118` are in use, change port mappings in `docker-compose.yml`.
- If you still see Ollama errors, check Admin Settings -> Connections and remove/disable stale Ollama entries if Ollama is not in use.
- Filter logs for old Ollama `11434` connection errors:
  ```bash
  docker compose logs --tail=500 openwebui | rg "11434|routers\\.ollama|/ollama/api/version|Connect call failed|Connection error"
  ```
- Filter logs for OpenAI verify/model-list timeout errors:
  ```bash
  docker compose logs --tail=500 openwebui | rg "verify_connection|/openai/verify|/openai/models/0|TimeoutError|Unexpected error| 500 "
  ```
