# Weekly CTF Writeup – JWT alg:none Authentication Bypass

## Overview

During a custom AI-assisted lab workflow using Hermes Agent, an authentication bypass vulnerability was identified and exploited successfully against a JWT-protected admin endpoint.

The issue was caused by improper JWT signature validation, allowing the use of the `alg:none` algorithm to forge administrative access tokens.

---

## Environment

### AI Workflow Stack
- Hermes Agent
- Claude Sonnet 4.6
- OpenRouter
- VPS-hosted autonomous workflow environment

### Target Type
- CTF / Lab Environment
- Authorized testing only

---

## Vulnerability

### Type
JWT Authentication Bypass (`alg:none`)

### Severity
High

### CWE
CWE-345: Insufficient Verification of Data Authenticity

---

## Discovery Process

Hermes Agent performed:
1. API endpoint enumeration
2. JWT structure analysis
3. Differential token comparison
4. Hypothesis generation
5. Authentication bypass testing

A hidden API path using a `/v1/` prefix was discovered during enumeration.

Example:
```http
/v1/admin/flag
```

The endpoint returned:
```json
{
  "message": "Admin access required"
}
```

---

## JWT Analysis

The observed JWT token:
- lacked an `isAdmin` field
- had no expiration (`exp`)
- used weak validation logic

Hermes identified the possibility that:
- signature validation might be disabled
- or `alg:none` could be accepted

---

## Exploitation

A forged JWT token was generated using:
```json
{
  "alg": "none",
  "typ": "JWT"
}
```

Payload:
```json
{
  "role": "admin",
  "isAdmin": true
}
```

No signature was added.

The forged token was supplied via:
```http
Authorization: Bearer <forged_token>
```

The server accepted the token and granted access to the protected endpoint.

---

## Result

The vulnerability allowed full authentication bypass and administrative access.

Flag retrieved successfully.

---

## Attack Chain Summary

1. Hidden `/v1/` API path discovered
2. Admin endpoint identified
3. JWT structure analyzed
4. Weak validation logic detected
5. `alg:none` attack attempted
6. Authentication bypass successful

---

## Root Cause

The backend improperly trusted JWT headers and failed to:
- enforce a secure signing algorithm
- validate signatures correctly
- reject unsigned tokens

---

## Remediation

### Recommended Fixes

- Explicitly enforce secure algorithms (`HS256`, `RS256`, etc.)
- Reject `alg:none`
- Always verify signatures server-side
- Use trusted JWT libraries only
- Validate claims properly (`exp`, `aud`, `iss`, etc.)

---

## AI-Assisted Workflow Notes

This workflow was executed using a custom-trained Hermes Agent configured for:
- HTB/CTF workflows
- autonomous reconnaissance
- exploit reasoning
- cybersecurity tooling
- vulnerability research

The most interesting aspect was not the vulnerability itself, but the agentic reasoning workflow:
- endpoint discovery
- token analysis
- exploit hypothesis generation
- validation chain execution

---

## Disclaimer

This writeup is for educational and authorized security research purposes only.

All testing was conducted in a legal lab/CTF environment with permission.
