# Load balancer for the Pond Planning API

Runs in front of two copies of the backend and sends each user to one of
them. While it runs you can switch where users go: spread them across both,
or send everyone to backend A or backend B. You switch from a control page
or with a single command. If a backend stops answering, its users move to
the other one automatically.

```
browser ──▶ frontend (/api) ──▶ load balancer :8000 ──┬──▶ backend A :8001
                                                      └──▶ backend B :8002
```

The frontend needs no changes: it already sends `/api` requests to port
8000, which is now the load balancer.

## Requirements

- Python 3.12 (tested with 3.12.3)
- `pip install -r requirements.txt` (Starlette, httpx, Uvicorn)
- Two backends to route to (the `backend` folder, started twice)

## Run

1. Start two backends, each in its own terminal, from the `backend` folder:

   ```bash
   uvicorn app.main:app --host 127.0.0.1 --port 8001 --workers 2
   uvicorn app.main:app --host 127.0.0.1 --port 8002 --workers 2
   ```

2. Start the load balancer from this folder:

   ```bash
   # macOS / Linux
   LB_BACKENDS="a=http://127.0.0.1:8001,b=http://127.0.0.1:8002" \
   uvicorn balancer:app --host 127.0.0.1 --port 8000
   ```

    These two backends are also the default, so `LB_BACKENDS` can be left
   out when they run on ports 8001 and 8002. Run the load balancer as a
   **single process**: don't add `--workers`, because the routing state is
   kept in memory. One process handles many users at once, since it only
   forwards requests.

3. Start the frontend as usual (`npm run dev` in `pond-frontend`).

To use two computers, start each backend with `--host 0.0.0.0` and list
their addresses, for example
`LB_BACKENDS="a=http://192.168.1.20:8001,b=http://192.168.1.21:8001"`.

## Switching backends

Open the control page at **http://127.0.0.1:8000/lb/**. Through the
frontend's development server, it is at http://localhost:5173/api/lb/.

![Control page](control-page.png)

Under **Send users to**, choose one of:

| Option | Mode name | What happens |
| --- | --- | --- |
| Keep each user on one backend | `sticky` (default) | Each user stays on one backend; new users go to the less busy one |
| Alternate between backends | `round_robin` | Requests take turns |
| Send each request to the least busy backend | `least_connections` | Each request goes where the fewest are in progress |
| Only backend A / Only backend B | `a` / `b` | Everyone goes to that backend; the other is used only while it is down |

Under **Backends**, *Take out of service* stops sending users to a backend,
for example before you restart or update it, and *Put back in service*
returns it. Changes apply to new requests at once; requests already running
finish where they are.

The same switches work from the command line:

```bash
curl -X POST http://127.0.0.1:8000/lb/mode -H "Content-Type: application/json" -d '{"mode": "b"}'
curl -X POST http://127.0.0.1:8000/lb/backends/a -H "Content-Type: application/json" -d '{"enabled": false}'
curl http://127.0.0.1:8000/lb/status
```

`sticky` is the default because it suits this app. A user's
`/findCatchment` call then reaches the backend that has just analysed their
file and still has it cached. In testing, that call took 17–43 ms instead of
a full rebuild of about a second.

## Settings

All settings are environment variables.

| Variable | Default | Meaning |
| --- | --- | --- |
| `LB_BACKENDS` | `a=http://127.0.0.1:8001,b=http://127.0.0.1:8002` | Backends as `name=url`, comma-separated. Without names they are called a, b, c… |
| `LB_MODE` | `sticky` | Mode at start-up: `sticky`, `round_robin`, `least_connections` or a backend name |
| `LB_ADMIN_TOKEN` | not set | Token required to view the status or switch backends (see below) |
| `LB_HEALTH_INTERVAL` | `5` | Seconds between health checks |
| `LB_HEALTH_PATH` | `/` | Path checked on each backend; must answer 200 |
| `LB_TIMEOUT` | `180` | Seconds to wait for a backend's answer |
| `LB_MAX_BODY_MB` | `110` | Largest upload accepted, in MB |

### Who can switch backends

Without `LB_ADMIN_TOKEN`, only requests from the computer the load balancer
runs on may use the control page and the `/lb/` admin API. The frontend's
development server runs on that same computer, so its users count as local.
**If other people use the app, set a token**:

```bash
LB_ADMIN_TOKEN="choose-a-long-random-value" uvicorn balancer:app --port 8000
```

The control page then asks for the token. The command line sends it as a
header: `curl -H "X-Admin-Token: choose-a-long-random-value" ...`.

## How it works

- **Health checks:** every backend gets `GET /` every few seconds. After two
  failed checks it is taken out of rotation, and after two passed checks it
  returns.
- **Failover:** if the load balancer can't connect to a backend, it marks it
  down and sends the same request to the other one at once, so users see no
  error. Retrying is safe because the analysis endpoints change nothing on
  the server. A backend that accepts a request but doesn't answer in time
  gets a 504; the request is not retried, which would only double the load.
- **Sticky users:** in `sticky` mode a cookie named `lb_backend` records each
  user's backend. If that backend goes down or is taken out of service, the
  user moves to the other one and the cookie is updated.
- **Headers:** every response carries `X-Backend` (which backend answered)
  and `X-Request-ID` (also written to the log). Backends receive
  `X-Forwarded-For`, `X-Forwarded-Proto` and `X-Forwarded-Host`.
- **Errors:** backend errors (400, 404, 413, …) pass through unchanged. The
  load balancer adds 413 for an upload over `LB_MAX_BODY_MB`, 502 when every
  backend failed, 503 when none is available, and 504 for a timeout.
- **Monitoring:** `GET /lb/health` answers 200 while at least one backend is
  available and 503 otherwise, for an uptime monitor.

## Tested

With two backends on one machine:

- Results through the load balancer were identical to calling a backend
  directly (93 ponds on the sample map).
- `round_robin` and `least_connections` alternated A, B, A, B.
- In `sticky` mode, new users alternated and each stayed on their backend.
- Switching to `a` or `b` moved all traffic at once.
- Taking A out of service sent everything to B.
- Killing backend B during traffic caused no failed requests. B rejoined
  2 seconds after restarting.
- With both backends out of service the balancer answered 503, and
  `/lb/health` turned 503.
- Six simultaneous analyses in `least_connections` split 3 and 3, all
  successful.
- The admin token, the upload limit (413) and the pass-through of backend
  errors (404, 400) behaved as described.
- The unchanged frontend's browser tests (file mode, second site, draw mode,
  mobile) passed through the load balancer.

## Limits

- The load balancer is a single process, so it is itself a single point of
  failure, and its counters and settings reset when it restarts. Set
  `LB_MODE` to choose the mode it starts in.
- Uploads are held in memory while being forwarded, at most
  `LB_MAX_BODY_MB` each.
- It serves plain HTTP. For HTTPS, put it behind a server such as Caddy or
  nginx, or terminate TLS in front of it.
- Each backend worker process has its own cache, so `sticky` raises the
  chance of a cache hit but can't guarantee one.

## Files

```
balancer.py        load balancer, health checks, admin API
control.html       control page served at /lb/
requirements.txt   pinned dependencies
```
