# Request Smuggling / Cache Poisoning Lab

A self-contained, **local-only** lab that demonstrates how an HTTP/1.1
**CL.TE request-smuggling** desync between a frontend proxy and a backend
server can be escalated into **cache poisoning**.

> [!WARNING]
> This lab is intentionally vulnerable. It binds only to `127.0.0.1` and is
> meant for learning and testing in an isolated environment. Do not expose any
> part of it to a network, and never copy this code into production.

## What it demonstrates

Two plain Python processes disagree about how to frame an HTTP/1.1 request:

| Component | Address | Framing rule |
| --- | --- | --- |
| Frontend proxy / cache (`proxy.py`) | `127.0.0.1:8083` | `Content-Length` wins; `Transfer-Encoding` is ignored |
| Origin / backend (`origin.py`) | `127.0.0.1:8084` | `Transfer-Encoding: chunked` wins over `Content-Length` |

That disagreement is a **CL.TE desync**. The frontend also strips the
`X-Internal` header from ordinary requests and caches `GET` responses by path.
The smuggled request bypasses both controls, so the origin answers
`GET /admin` with the admin flag and the proxy stores that response under `/`.
Every later visitor to `/` is then served the admin response from the cache.

See [ATTACK.md](ATTACK.md) for the full step-by-step walkthrough.

## Requirements

- Python 3 (no third-party packages)
- A POSIX shell (`bash`) for `start.sh` / `stop.sh`

No Docker, system services, database, or restart policy is involved.

## Quick start

`start.sh` runs in the foreground, so use two terminals:

**Terminal 1 — start the lab**

```bash
cd request-smuggling-lab
./start.sh
# Frontend/cache -> http://127.0.0.1:8083/
# Origin (internal) -> 127.0.0.1:8084
```

**Terminal 2 — run the exploit**

```bash
cd request-smuggling-lab
python3 exploit.py
```

The exploit issues `PURGE /` first, so it is **repeatable** — the in-memory
cache is cleared on every run.

**Stop the lab** from either terminal (or press `Ctrl-C` in terminal 1):

```bash
./stop.sh
```

### Expected result

`exploit.py` prints three responses. The important lines are:

```
--- Victim request gets the smuggled admin response ---
ADMIN: FLAG{request-smuggling-admin-exposed}

--- Fresh victim - poisoned cache ---
x-cache: HIT
ADMIN: FLAG{request-smuggling-admin-exposed}
```

The first response (to the attacker's `POST`) is the normal origin page. The
second is the smuggled `/admin` response landing on the victim's request. The
third proves that a **fresh connection** now receives the poisoned cache entry.

## How it works

1. The attacker sends `POST /` with **both** `Content-Length` and
   `Transfer-Encoding: chunked`. Its body contains a `0\r\n\r\n` chunk
   terminator followed by a smuggled `GET /admin` carrying `X-Internal: 1`.
2. The frontend reads `Content-Length` bytes and forwards the whole blob to the
   origin on its shared, keep-alive connection.
3. The origin stops at the chunk terminator and treats the remaining
   `GET /admin ...` as a new request.
4. The victim's `GET /` is sent next on that same connection. The origin replies
   to the smuggled `/admin` first, and the proxy caches that response under the
   key `("GET", "/")`.
5. A fresh visitor to `/` gets `X-Cache: HIT` with the admin flag instead of the
   normal page.

## Project layout

| Path | Purpose |
| --- | --- |
| `proxy.py` | Frontend cache/proxy — frames by `Content-Length` |
| `origin.py` | Backend — frames by `Transfer-Encoding: chunked` |
| `http_helpers.py` | Shared, deliberately minimal HTTP/1.1 framing helpers |
| `exploit.py` | CL.TE smuggling + cache poisoning proof of concept |
| `ATTACK.md` | Detailed attack walkthrough and remediation guidance |
| `start.sh` / `stop.sh` | Start/stop the two lab processes |
| `origin.log` / `proxy.log` | stdout/stderr of the running processes |
| `.server.pid` | PIDs of the running lab processes (managed by the scripts) |

## Notes and limitations

- This is **infrastructure-level**, not Laravel application code. In a real
  deployment the origin could be a Laravel app behind nginx/CDN.
- The cache is an in-memory dict, and the frontend caches `GET` responses by
  path only — no query string, headers, cookies, or scheme/host in the key.
- The framing parsers are intentionally minimal: they exist to make the CL/TE
  mismatch visible, not to be production HTTP servers.
- Connections are plain HTTP/1.1 with keep-alive; there is no TLS.
- The origin log may show harmless `ConnectionResetError` stack traces when the
  exploit closes its sockets. The smuggling still succeeds.

## Troubleshooting

- **`Lab already running (PID ...)`** — a previous `start.sh` is still alive.
  Run `./stop.sh` first, or use the printed PIDs.
- **Port already in use** — make sure nothing else is listening on `8083` or
  `8084` (for example: `ss -ltnp | grep -E '8083|8084'`).
- **No admin flag / no `x-cache: HIT`** — confirm both processes are running
  (`ps -ef | grep -E 'proxy.py|origin.py'`), check `proxy.log` and `origin.log`,
  then re-run `python3 exploit.py`.

## Roadmap

The lab currently ships in its vulnerable state (Phase 1). Planned Phase 2
hardening, outlined in [ATTACK.md](ATTACK.md), includes rejecting requests that
carry both framing headers, normalizing framing at a single layer, using
HTTP/2 end-to-end where possible, and widening the cache key. The fix is
**not applied** in this repository yet; re-run `python3 exploit.py` after it is.
