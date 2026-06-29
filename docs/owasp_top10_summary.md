# OWASP Top 10 – Web Application Vulnerabilities Summary

**Author:** Harshit
**Program:** MCA – Cybersecurity Specialization, Jain University Bangalore
**Task:** Week 2 – Task 1 | Cybersecurity Internship
**Reference:** OWASP Top 10 (2021 Edition)
**Date:** June 2025

---

## What is OWASP?

The **Open Web Application Security Project (OWASP)** is a non-profit foundation dedicated to improving software security. Their **Top 10** list is the globally recognized standard for the most critical web application security risks, updated periodically based on real-world breach data and expert consensus.

The OWASP Top 10 is used by:
- Developers to write secure code
- Security teams to prioritize vulnerabilities
- Organizations to define security policies
- Penetration testers to guide assessments

---

## OWASP Top 10 (2021) – Overview Table

| Rank | Category | Common CWEs | Previously |
|------|---------|-------------|-----------|
| A01 | Broken Access Control | CWE-200, CWE-284, CWE-352 | Was A05 (2017) |
| A02 | Cryptographic Failures | CWE-259, CWE-327, CWE-331 | Was A03 (Sensitive Data Exposure) |
| A03 | Injection | CWE-79, CWE-89, CWE-73 | Was A01 (2017) |
| A04 | Insecure Design | CWE-209, CWE-256, CWE-501 | New in 2021 |
| A05 | Security Misconfiguration | CWE-16, CWE-611 | Was A06 (2017) |
| A06 | Vulnerable & Outdated Components | CWE-1104 | Was A09 (2017) |
| A07 | Identification & Auth Failures | CWE-287, CWE-384 | Was A02 (2017) |
| A08 | Software & Data Integrity Failures | CWE-494, CWE-502 | New in 2021 |
| A09 | Security Logging & Monitoring Failures | CWE-223, CWE-778 | Was A10 (2017) |
| A10 | Server-Side Request Forgery (SSRF) | CWE-918 | New in 2021 |

---

## A01 – Broken Access Control 🔴

**Severity: Critical | Most Common in 2021**

### What it is
Access control enforces policies that restrict users to only what they are permitted to do. Broken access control occurs when these restrictions are not properly enforced, allowing users to act outside their intended permissions.

### How it occurs
- Accessing other users' accounts by modifying URL parameters (IDOR)
- Viewing or editing someone else's data without authorization
- Accessing admin pages without being an admin
- Elevation of privilege (acting as admin without being logged in as one)

### Example
```
# User accesses their own profile:
https://example.com/user/profile?id=1001

# Attacker changes the ID to access another user's data:
https://example.com/user/profile?id=1002  ← Insecure Direct Object Reference (IDOR)
```

### Prevention
- Deny access by default — only allow what is explicitly permitted
- Implement server-side access control on every endpoint (never trust the client)
- Log and alert on access control failures
- Disable directory listing on web servers
- Use unique, unpredictable identifiers (UUIDs) instead of sequential IDs

---

## A02 – Cryptographic Failures 🔴

**Severity: Critical**

### What it is
Failures related to cryptography that expose sensitive data. Previously called "Sensitive Data Exposure," this category now focuses on the root cause — weak or missing cryptography.

### How it occurs
- Transmitting data in plaintext (HTTP instead of HTTPS)
- Using weak or outdated encryption algorithms (MD5, SHA-1, DES)
- Weak cryptographic key generation or management
- Not encrypting sensitive data at rest (database passwords, credit card numbers)
- Using hardcoded passwords or keys in source code

### Example
```python
# ❌ WRONG – MD5 is cryptographically broken
import hashlib
hashed = hashlib.md5(password.encode()).hexdigest()

# ✅ CORRECT – bcrypt with salt
import bcrypt
hashed = bcrypt.hashpw(password.encode(), bcrypt.gensalt(rounds=12))
```

### Prevention
- Classify all data and apply appropriate protections based on sensitivity
- Do not store sensitive data unnecessarily
- Encrypt all data at rest and in transit (TLS 1.2+, AES-256)
- Use strong, modern hashing algorithms for passwords (bcrypt, Argon2)
- Never hardcode keys or passwords in source code

---

## A03 – Injection 🔴

**Severity: Critical | Includes SQL Injection & XSS**

### What it is
Injection flaws occur when untrusted data is sent to an interpreter (SQL engine, OS shell, LDAP server) as part of a command or query. The attacker's hostile data can trick the interpreter into executing unintended commands.

### Subtypes
| Type | Target | Example Payload |
|------|--------|----------------|
| SQL Injection | Database | `' OR 1=1--` |
| Cross-Site Scripting | Browser DOM | `<script>alert(1)</script>` |
| Command Injection | OS Shell | `; rm -rf /` |
| LDAP Injection | Directory Services | `*)(uid=*))(|(uid=*` |
| XXE | XML Parser | External entity reference |

### SQL Injection Example
```sql
-- Vulnerable query (string concatenation):
SELECT * FROM users WHERE email = '[input]' AND password = '[input]';

-- Attacker input: ' OR 1=1--
-- Resulting query:
SELECT * FROM users WHERE email = '' OR 1=1--' AND password = '';
-- 1=1 is always TRUE → bypass login
```

### Prevention
- Use parameterized queries / prepared statements
- Use an ORM (Sequelize, Hibernate) that avoids direct SQL construction
- Validate and whitelist input on the server side
- Apply least privilege to database accounts
- Use a WAF as an additional layer

---

## A04 – Insecure Design 🟠

**Severity: High | New in 2021**

### What it is
A new category in 2021 focusing on risks related to design and architectural flaws. Insecure design cannot be fixed by a perfect implementation — the design itself is fundamentally flawed.

### How it occurs
- No threat modeling during the design phase
- Business logic flaws (e.g., allowing negative quantities in a shopping cart)
- No rate limiting designed into authentication flows
- Designing APIs that expose too much data by default

### Example
A flight booking system that allows users to select their seat does not validate that the seat belongs to their booking. Another user can take any seat by crafting a request — a business logic flaw baked into the design.

### Prevention
- Conduct threat modeling during the design phase
- Use secure design patterns and reference architectures
- Integrate security requirements into user stories
- Perform unit and integration tests to validate security controls
- Apply the principle of least privilege by design

---

## A05 – Security Misconfiguration 🟠

**Severity: High | Most Commonly Found**

### What it is
Security misconfiguration is the most commonly found vulnerability. It occurs when security settings are defined, implemented, or maintained incorrectly — or simply left at insecure defaults.

### How it occurs
- Default credentials not changed (admin/admin on routers, databases)
- Unnecessary features enabled (debug mode, default accounts)
- Error messages revealing stack traces and internal paths
- Missing security headers (CSP, X-Frame-Options, HSTS)
- Cloud storage buckets misconfigured as public (AWS S3)
- Directory listing enabled on web servers

### Example
```
# Developer left debug mode on in production Django app:
DEBUG = True  # Reveals full stack traces, source code paths, environment variables

# Or: Default MongoDB with no authentication:
mongod --bind_ip 0.0.0.0  # No auth, accessible from anywhere
```

### Prevention
- Establish a hardened, repeatable configuration process for all environments
- Disable or remove all unnecessary features, components, and accounts
- Apply security headers (use Helmet.js for Node, SecurityMiddleware for Django)
- Regularly review and audit cloud storage permissions
- Automate configuration review with tools like AWS Config, CIS Benchmarks

---

## A06 – Vulnerable & Outdated Components 🟠

**Severity: High**

### What it is
Using software components (libraries, frameworks, plugins) with known vulnerabilities. This is how WannaCry (EternalBlue) and the Equifax breach occurred.

### How it occurs
- Not knowing the versions of all components in use
- Not subscribing to security advisories for dependencies
- Not testing compatibility of updated libraries (leads to update avoidance)
- Using abandoned or unmaintained open-source libraries

### Prevention
- Maintain a software bill of materials (SBOM) for all dependencies
- Use tools like OWASP Dependency-Check, Snyk, or GitHub Dependabot
- Remove unused dependencies and features
- Monitor CVE databases for newly disclosed vulnerabilities
- Apply patches promptly — set an SLA (e.g., critical CVEs patched within 48 hours)

---

## A07 – Identification & Authentication Failures 🔴

**Severity: Critical | Was A02 in 2017**

### What it is
Weaknesses in authentication and session management that allow attackers to compromise passwords, keys, or session tokens, or exploit other implementation flaws to assume other users' identities.

### How it occurs
- Allowing brute force or credential stuffing attacks (no rate limiting)
- Permitting weak or commonly used passwords
- Weak credential recovery processes (knowledge-based answers)
- Storing passwords in plaintext or with weak hashing (MD5)
- Missing or ineffective MFA
- Exposing session IDs in URLs
- Not invalidating sessions after logout

### Prevention
- Implement MFA wherever possible
- Do not allow weak or breached passwords (check against HaveIBeenPwned)
- Limit or delay failed login attempts; alert on brute force attempts
- Use a secure server-side session manager that generates high-entropy session IDs
- Invalidate sessions on the server side after logout

---

## A08 – Software & Data Integrity Failures 🟠

**Severity: High | New in 2021**

### What it is
Code and infrastructure that does not protect against integrity violations. Includes scenarios where an application relies on plugins, libraries, or modules from untrusted sources, repositories, or CDNs — and insecure deserialization.

### How it occurs
- Using unsigned or unverified software updates
- Pulling dependencies from untrusted CDNs or repositories
- Auto-update functionality without integrity verification
- Deserializing untrusted data without validation (can lead to RCE)
- Insecure CI/CD pipelines that allow unauthorized code injection

### Real-World Example
The **SolarWinds attack (2020):** Malicious code was injected into a legitimate software update, affecting 18,000 organizations including US government agencies.

### Prevention
- Use digital signatures to verify software and data
- Ensure dependencies are consumed from trusted repositories
- Use software composition analysis tools
- Review CI/CD pipeline configurations for unauthorized access
- Implement integrity checking for serialized data

---

## A09 – Security Logging & Monitoring Failures 🟠

**Severity: High**

### What it is
Insufficient logging and monitoring — including missing detection, inadequate monitoring, and failure to respond — allows attackers to achieve their objectives undetected. Most breaches take **over 200 days** to detect.

### How it occurs
- Login failures and other security events not logged
- Warnings and errors generate no or inadequate log messages
- Logs not monitored for suspicious activity
- Logs only stored locally (attackers can delete them)
- No alerting thresholds or incident response plans

### Prevention
- Log all authentication events (success and failure), access control failures, and input validation failures
- Ensure log format is consumable by SIEM solutions
- Protect logs with append-only storage — prevent tampering
- Establish effective monitoring and alerting
- Establish and test an incident response plan

---

## A10 – Server-Side Request Forgery (SSRF) 🟠

**Severity: High | New in 2021**

### What it is
SSRF flaws occur when a web application fetches a remote resource without validating the user-supplied URL. This allows attackers to force the server to make requests to internal services that should not be accessible.

### How it occurs
- Web applications that fetch URLs on behalf of users (URL preview, PDF generators, webhooks)
- No validation of the URL's destination
- Cloud metadata endpoints accessible from the application server

### Example
```
# Normal usage:
POST /fetch-preview
url=https://legitimate-website.com/image.jpg

# SSRF attack — attacker requests internal metadata:
POST /fetch-preview
url=http://169.254.169.254/latest/meta-data/iam/security-credentials/
# AWS metadata endpoint returns cloud credentials!
```

### Prevention
- Sanitize and validate all client-supplied input data
- Enforce URL allowlists — only permit specific domains/IPs
- Do not forward raw HTTP responses to clients
- Disable HTTP redirections where not required
- Block access to internal/metadata IP ranges at the network level

---

## Summary: Prevention Principles

| Principle | Addresses |
|-----------|----------|
| Input validation & parameterized queries | A03 Injection |
| Strong encryption & key management | A02 Cryptographic Failures |
| Deny-by-default access control | A01 Broken Access Control |
| MFA + rate limiting + strong passwords | A07 Auth Failures |
| Security headers + minimal configuration | A05 Misconfiguration |
| Dependency scanning & patching | A06 Outdated Components |
| Centralized logging + SIEM + alerting | A09 Logging Failures |
| Threat modeling in design phase | A04 Insecure Design |
| Signed updates + secure CI/CD | A08 Integrity Failures |
| URL allowlisting + network controls | A10 SSRF |

---

*Document prepared as part of Cybersecurity Internship – Week 2, Task 1*
*Reference: https://owasp.org/Top10/*
