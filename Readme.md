# Open WebUI Local Setup

This repository runs Open WebUI with a local HTTP proxy (`privoxy`) that forwards traffic through a SOCKS5 upstream.

## What This Stack Does

- Runs Open WebUI in Docker (`openwebui` service)
- Runs Privoxy in Docker (`privoxy` service)
- Forces Open WebUI outbound HTTP(S) requests through Privoxy
- Makes Privoxy forward those requests to a SOCKS5 endpoint at `host.docker.internal:4567`

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
4. Privoxy then forwards all requests using:
   - `forward-socks5t / host.docker.internal:4567 .`
5. `forward-socks5t` means DNS/hostname resolution is done through SOCKS (remote DNS), not locally.

## LLM Endpoint In This Setup

Open WebUI is configured with:

- `OPENAI_API_BASE_URL=http://nimbus.chpc.utah.edu:44142/v1`

So API calls from Open WebUI are intended to go:

- `openwebui` -> `privoxy` -> SOCKS5 at `host.docker.internal:4567` -> remote endpoint (`nimbus.chpc.utah.edu:44142`)

## NO_PROXY Behavior

These hosts bypass the proxy from inside the Open WebUI container:

- `localhost`
- `127.0.0.1`
- `::1`
- `host.docker.internal`
- `172.17.0.1`

This is controlled by:

- `NO_PROXY=localhost,127.0.0.1,::1,host.docker.internal,172.17.0.1`

## Current Known Note

In `privoxy/config`, there is a note that selective forwarding by host (for just `nimbus.chpc.utah.edu`) was tested on March 12, 2026 and did not behave as expected, so the config currently forwards all requests over SOCKS.

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
