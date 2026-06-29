# OWASP Juice Shop – Safe Demo Lab Findings

**Author:** Harshit
**Task:** Week 2 – Task 1 | Cybersecurity Internship
**Environment:** OWASP Juice Shop v15.x (local Docker, `http://localhost:3000`)
**Date:** June 2025

> ⚠️ **Disclaimer:** All findings documented here were discovered on OWASP Juice Shop — a deliberately vulnerable application built specifically for security education. No real-world or production systems were accessed. This is a purely educational exercise conducted in an isolated local environment.

---

## Lab Setup

```bash
# Run Juice Shop locally via Docker
docker pull bkimminich/juice-shop
docker run --rm -p 3000:3000 bkimminich/juice-shop

# Access at: http://localhost:3000
```

**Tools used:**
- Mozilla Firefox + DevTools
- Burp Suite Community Edition (HTTP interception)
- jwt.io (JWT decoder)

---

## Finding 1 – SQL Injection on Login Page

| Field | Detail |
|-------|--------|
| Location | `http://localhost:3000/#/login` |
| Input field | Email |
| Payload | `' OR 1=1--` |
| OWASP Category | A03 – Injection |
| Severity | 🔴 Critical |

**Reproduction Steps:**
1. Navigate to `http://localhost:3000/#/login`
2. In Email field, enter: `' OR 1=1--`
3. In Password field, enter any value (e.g., `abc`)
4. Click **Log In**

**Observed Result:**
- Application logged in as `admin@juice-sh.op`
- Admin account accessed without valid credentials
- Browser redirected to user dashboard showing admin email

**Root Cause:**
The login query concatenates raw user input into SQL without parameterization, allowing the injected `OR 1=1` condition to always evaluate as true, bypassing authentication entirely.

**Screenshot:** `screenshots/sql_injection_login.png`

**Recommended Fix:**
```javascript
// Replace string concatenation with parameterized query
db.execute('SELECT * FROM Users WHERE email = ? AND password = ?', [email, hash]);
```

---

## Finding 2 – Reflected XSS in Search Bar

| Field | Detail |
|-------|--------|
| Location | `http://localhost:3000/#/search` |
| Input field | Search bar (q parameter) |
| Payload | `<iframe src="javascript:alert('XSS')">` |
| OWASP Category | A03 – Injection (XSS) |
| Severity | 🔴 High |

**Reproduction Steps:**
1. Click the search icon (🔍) in the top navigation bar
2. Enter the following payload and press Enter:
```html
<iframe src="javascript:alert('XSS')">
```
3. Observe the browser alert dialog appear

**Observed Result:**
- JavaScript alert box appeared with text "XSS"
- Confirmed that user input is reflected back into the DOM without sanitization
- The `javascript:` URI scheme executed in the browser context

**Root Cause:**
The Angular search component reflects the `q` URL parameter into the DOM using an unsafe binding method, allowing HTML/JavaScript injection without encoding or sanitization.

**Screenshot:** `screenshots/xss_alert_popup.png`

**Recommended Fix:**
```typescript
// Use Angular's DomSanitizer instead of raw innerHTML binding
this.safeQuery = this.sanitizer.sanitize(SecurityContext.HTML, rawInput);
```

---

## Finding 3 – Weak JWT Token Secret

| Field | Detail |
|-------|--------|
| Location | HTTP Authorization header / Local Storage |
| Method | JWT decode + payload inspection |
| Tool | jwt.io decoder |
| OWASP Category | A07 – Identification & Auth Failures |
| Severity | 🔴 High |

**Reproduction Steps:**
1. Log in with any valid Juice Shop account (register first if needed)
2. Open Firefox DevTools (F12) → Application → Local Storage → `http://localhost:3000`
3. Copy the full `token` value
4. Open `https://jwt.io` in another tab
5. Paste the token into the "Encoded" box on the left

**Observed Result:**
JWT successfully decoded without any key, revealing:
```json
{
  "data": {
    "email": "test@harshit.com",
    "role": "customer",
    "lastLoginIp": "0.0.0.0",
    "loggedInAs": "test@harshit.com"
  },
  "iat": 1718000000,
  "exp": 1718086400
}
```

The `role` field is stored directly in the JWT payload. The known weak secret (`secret`) allows re-signing a modified token with `"role": "admin"`.

**Root Cause:**
- JWT signed with hardcoded weak secret (`secret`)
- Sensitive privilege information (`role`) stored in the token payload
- Server trusts the token's `role` field without cross-referencing the database

**Screenshot:** `screenshots/jwt_decoded.png`

**Recommended Fix:**
```javascript
// 1. Use a strong secret from environment variables
const token = jwt.sign({ userId: user.id }, process.env.JWT_SECRET, { expiresIn: '15m' });

// 2. Never trust role from JWT — always fetch from DB
const decoded = jwt.verify(token, process.env.JWT_SECRET);
const user = await User.findById(decoded.userId);  // role comes from DB
```

---

## Finding 4 – No Rate Limiting on Login Endpoint

| Field | Detail |
|-------|--------|
| Location | `POST /rest/user/login` |
| Method | Repeated failed login attempts |
| OWASP Category | A07 – Identification & Auth Failures |
| Severity | 🟠 High |

**Reproduction Steps:**
1. Go to the login page
2. Enter any email and a wrong password
3. Click Log In repeatedly (10+ times)

**Observed Result:**
- No lockout after multiple failures
- No CAPTCHA appeared
- No increasing delay between attempts
- Error message immediately returned each time

**Root Cause:**
The login endpoint has no brute force protection — an automated tool like Hydra or Burp Intruder could send thousands of password attempts per minute.

**Recommended Fix:**
```javascript
const rateLimit = require('express-rate-limit');
app.use('/rest/user/login', rateLimit({
    windowMs: 15 * 60 * 1000,
    max: 10,
    message: 'Too many attempts. Try again in 15 minutes.'
}));
```

---

## Summary of Findings

| # | Vulnerability | OWASP | Severity | Fix |
|---|--------------|-------|----------|-----|
| 1 | SQL Injection – Login bypass | A03 | 🔴 Critical | Parameterized queries |
| 2 | Reflected XSS – Search bar | A03 | 🔴 High | Output encoding + CSP |
| 3 | Weak JWT secret + role in payload | A07 | 🔴 High | Strong secret + DB role check |
| 4 | No rate limiting on login | A07 | 🟠 High | Rate limiter middleware |

---

## Key Takeaways from Lab

1. **Never trust user input** — every input field is a potential attack surface
2. **Authentication is not just passwords** — JWT secrets, session management, and rate limiting all matter
3. **Juice Shop makes it easy to see WHY** these vulnerabilities exist — the code is readable and the flaws are intentional teaching tools
4. **Fix is always simpler than the vulnerability** — one line of parameterization prevents SQLi; one CSP header blocks XSS

---

*Lab findings documented as part of Cybersecurity Internship – Week 2, Task 1*
*OWASP Juice Shop: https://owasp.org/www-project-juice-shop/*
