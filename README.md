# Fanon Comic Assignment — Backend Companion

Thanks for showing interest in joining Fanon's App team. This repo serves as a companion for your assignment: a comic catalogue API, as well as an MCP server for your agent to query (if you choose to use one)

The only prerequisite to run this is a Docker engine, so it should work on your platform irrespective of your OS or CPU architecture. 

Once started, two containers will boot up:

| Service | Address | What it is |
| --- | --- | --- |
| `api` | `http://localhost:3001` | The REST API your app talks to |
| `mcp` | `http://localhost:3002/mcp` | The API documentation, over MCP, for your agent |

---

## Quick start

```sh
docker compose up -d
```

Then confirm the API is answering:

```sh
curl http://localhost:3001/health
```

```json
{"status":"HEALTHY"}
```

Both services should read `healthy`:

```sh
docker compose ps
```

`.env` is optional. Every setting already has the same default baked into `docker-compose.yml`, so `docker compose up -d` works with no `.env` at all. Copy `.env.example` to `.env` only if you want to change ports, change the log level, or inject artificial latency to test your loading states.

Logs: `docker compose logs -f api`. Stop: `docker compose down`.

---

## Reaching the API from your app

The server binds `0.0.0.0`, so an emulator or a physical device can reach it, but the address of "the host machine" is different in each case. Here's a handy guide:

| Where your app runs | Base URL |
| --- | --- |
| Host machine, or a web build | `http://localhost:3001` |
| iOS Simulator | `http://localhost:3001` |
| Android emulator | `http://10.0.2.2:3001` |
| Physical device on the same network | `http://<host-LAN-IP>:3001` |

An Android emulator's `localhost` is the emulator itself; `10.0.2.2` is the emulator's alias for the host. A physical device has no route to the host's loopback at all, so it needs the host's LAN address.

Find that address:

| Host OS | Command |
| --- | --- |
| macOS | `ipconfig getifaddr en0` |
| Windows | `ipconfig` — read the `IPv4 Address` line |
| Linux | `hostname -I` |

Then prove it from the host before you suspect your app:

```sh
curl http://192.168.1.24:3001/health
```

If that answers from the host but the phone still cannot connect, it is the host firewall or client isolation on the Wi-Fi network.

---

## Endpoints

| Method | Path | Returns |
| --- | --- | --- |
| GET | `/stories` | Story feed |
| GET | `/stories/{storyId}` | More details about a single story |
| GET | `/chapters?story={storyId}` | Chapter listing for one story |
| GET | `/chapters/{chapterId}` | Reader payload: the pages of one chapter |
| GET | `/health` | Liveness of the process and its database |
| GET | `/openapi.json` | The full contract as an OpenAPI document |
| GET | `/docs` | The same contract, browsable in a page |

The two list endpoints accept `limit` (a value above the cap is clamped rather than rejected) and `cursor` (the checkpoint from the previous response), and answer with one envelope:

```json
{ "page": [ ], "hasMore": true, "continueCursor": "opaque string, or null when hasMore is false" }
```

The documents live under `page`. The two `{id}` endpoints read no query parameters.

Errors use a single envelope too. Branch on `code`; `message` was for my development and its wording may change.

```sh
curl "http://localhost:3001/stories?limit=0"
```

```json
{"error":{"code":"INVALID_LIMIT","message":"limit must be at least 1, got 0"}}
```

| Status | `code` |
| --- | --- |
| 400 | `INVALID_LIMIT`, `INVALID_CURSOR`, `MISSING_PARAMETER` |
| 404 | `NOT_FOUND` |
| 405 | `METHOD_NOT_ALLOWED` |
| 500 | `INTERNAL` |

**The full contract lives in the MCP server**, not in this file: every parameter, every
field of every response shape, sample payloads, and the exact value formats. Wire it up
and ask your agent instead of working from the table above.

Not driving an agent? The same contract is served as OpenAPI at
`http://localhost:3001/openapi.json`, and browsable in your browser at
`http://localhost:3001/docs`.

---

## Wiring the documentation MCP server into your agent

The `mcp` service serves the API documentation over MCP at `http://localhost:3002/mcp`.
Server identity `fanon-assignment-docs` v1.0.0, protocol `2025-06-18`, 8 tools and 20
documentation resources.

### Claude Code

```sh
claude mcp add --transport http fanon-docs http://localhost:3002/mcp
```

### Cursor, Cline, Windsurf, and anything else taking `mcpServers` JSON

Cursor reads `.cursor/mcp.json` (or the global `~/.cursor/mcp.json`); Cline and Windsurf
each have their own MCP settings file. The block is the same:

```json
{
  "mcpServers": {
    "fanon-docs": {
      "type": "http",
      "url": "http://localhost:3002/mcp"
    }
  }
}
```

### Codex

In `~/.codex/config.toml`:

```toml
[mcp_servers.fanon-docs]
url = "http://localhost:3002/mcp"
```

### Anything that speaks only stdio

The same server runs as a stdio process inside the container:

```json
{
  "mcpServers": {
    "fanon-docs": {
      "command": "docker",
      "args": ["exec", "-i", "app-assignment-backend-mcp-1", "/app/mcpd", "-stdio"]
    }
  }
}
```

The container name is `<compose-project>-mcp-1`, and the project name is pinned to
`app-assignment-backend` in `docker-compose.yml`, so it is stable. The stack has to be
up. From a shell inside this repo, `docker compose exec -T mcp /app/mcpd -stdio` is the
equivalent.

### Testing whether it is up

```sh
curl http://localhost:3002/healthz
```

```json
{"status":"HEALTHY"}
```

Use `/healthz` for that check, not `/mcp`. A plain browser or `curl` GET of `/mcp` answers 4xx by design: the streamable-HTTP transport accepts POST, and refuses a GET that does not offer `Accept: text/event-stream`. Depending on the `Accept` header your client sends you will see `405 Method Not Allowed` or `400 Bad Request`,  both mean the server is running.

Once it is wired, questions like "what query parameters does GET /chapters take?" or "show me a sample chapter payload" get answered from the documentation rather than guessed.

---

## Data notes

- Every documented key is always present, and `continueCursor` is the only value that is ever `null`.
- CORS is open. Responses are gzipped when the request offers `Accept-Encoding: gzip`.
  Every response carries `X-Request-Id`; send your own value in that header and it is adopted and echoed back, so your log line and the server's line match.

**Images come from `picsum.photos`.** This container does not serve them. I've stored URLs, and your device fetches the bytes from picsum that is rate-limited and entirely outside this companion's control. A burst of image requests can come back with code `429` or time out. That is picsum, not an API failure, and `/health` will still report `HEALTHY` throughout. Image URLs are deterministic: a given page always has the same URL, so ordinary HTTP caching applies to them.

---

## Refreshing the image

```sh
docker compose pull && docker compose down -v && docker compose up -d
```

---

## Troubleshooting

**`bind: address already in use` on `up`.** Something else already owns 3001 or 3002.
Set `API_PORT` / `MCP_PORT` in `.env`. They move only the **host** side of the port mapping, inside their containers the servers always listen on 3001 and 3002; and put the new host port in your app's base URL.

**`.env` looks like it is being ignored.** A shell environment variable of the same name takes precedence over `.env`. 
Check your shell for an exported `API_PORT`, `MCP_PORT`, `IMAGE_TAG` or `LOG_LEVEL` before editing the file again. `docker compose config` prints the fully resolved configuration, which is the quickest way to see what Compose actually
received.

**`api` restart-loops with `file is not a database (26)`.**
Stale volume, see [Refreshing the image](#refreshing-the-image).

**Cannot reach the API from a phone or an emulator.**
See [Reaching the API from your app](#reaching-the-api-from-your-app). `localhost` does not mean the host machine from inside an emulator or on a device.

**`GET /mcp` answers 400 or 405.**
Expected. See [Testing whether it is up](#testing-whether-it-is-up).

**Images do not load, but `/health` is `HEALTHY`.**
Picsum rate limiting. See [Data notes](#data-notes).

---
