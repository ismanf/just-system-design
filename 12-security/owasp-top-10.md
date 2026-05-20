# OWASP Top 10

> **TL;DR** — The **OWASP Top 10** is the industry's most-referenced list of the most critical web application security risks, updated every few years by the Open Worldwide Application Security Project. The current 2021 edition (next major update due 2024–2025) ranks: broken access control, cryptographic failures, injection, insecure design, security misconfiguration, vulnerable & outdated components, authentication failures, software & data integrity failures, security logging failures, and SSRF. It's not a checklist — it's a syllabus. Every engineer should know what each item *means*, how it manifests in code, and what the standard mitigations are. Most real breaches map to one of these ten categories.

---

## 1. The List (2021 Edition)

| # | Category | Renamed/merged from earlier |
| --- | --- | --- |
| **A01** | Broken Access Control | (up from #5) |
| **A02** | Cryptographic Failures | (was "Sensitive Data Exposure") |
| **A03** | Injection | (down from #1; now includes XSS) |
| **A04** | Insecure Design | (new — design-level mistakes) |
| **A05** | Security Misconfiguration | (incl. old "XXE") |
| **A06** | Vulnerable & Outdated Components | |
| **A07** | Identification & Authentication Failures | (was "Broken Authentication") |
| **A08** | Software & Data Integrity Failures | (new — supply chain, deserialization) |
| **A09** | Security Logging & Monitoring Failures | |
| **A10** | Server-Side Request Forgery (SSRF) | (new — promoted from CWE) |

OWASP also publishes an **API Security Top 10** (separate list, different priorities — BOLA, broken auth, etc.) and a **Mobile Top 10**. For server-side web app work, the main Top 10 is the relevant reference.

---

## 2. A01 — Broken Access Control

**What it means:** users can do things they shouldn't — view other users' data, escalate privileges, modify records they don't own.

**Manifestations:**
- **IDOR** (Insecure Direct Object Reference): `GET /invoices/42` returns invoice 42 whether or not it belongs to you.
- **Missing role checks**: every endpoint should be authorized; some aren't.
- **JWT tampering**: a client edits `role: admin` in the claims and the server fails to verify.
- **Path manipulation**: `/users/../admin/secrets`.
- **Force-browsing**: `/admin/dashboard` accessible without being an admin.
- **CORS misconfig** allowing arbitrary origins to read authenticated responses.

**Mitigations:**
- Authorize **per resource**, not just per endpoint. "Logged-in" ≠ "allowed to read this thing."
- Centralize policy in a PDP (OPA, Cedar, SpiceDB) rather than scattering `if user.role == "admin"`.
- Default-deny: routes opt-in to public, not the reverse.
- Reject any client-side enforcement; the server is the only authority.
- See [RBAC, ABAC, ReBAC →](./access-control.md).

---

## 3. A02 — Cryptographic Failures

**What it means:** sensitive data isn't protected — wrong algorithms, missing TLS, hardcoded keys, predictable randomness.

**Manifestations:**
- HTTP (not HTTPS) for any sensitive page.
- Passwords stored unhashed or hashed with MD5/SHA-256.
- TLS misconfigured: old versions enabled, weak ciphers, expired certs.
- Hardcoded keys in source code.
- Using `Math.random()` for security tokens (it's not cryptographic).
- AES-ECB mode (leaks patterns).
- Plaintext data in logs or backups.

**Mitigations:**
- HTTPS everywhere; HSTS; modern TLS config.
- AES-256-GCM (or ChaCha20-Poly1305) for symmetric.
- Ed25519 / ECDSA P-256 for signatures.
- Argon2id / bcrypt for passwords.
- CSPRNGs for tokens (`crypto.randomBytes`, `secrets.token_urlsafe`).
- Keys in a KMS; rotate routinely.
- See [Encryption at Rest & In Transit →](./encryption.md) and [Password Storage →](./password-storage.md).

---

## 4. A03 — Injection

**What it means:** untrusted data is mixed with code or queries, allowing attackers to execute their own commands.

**Manifestations:**
- **SQL injection**: `"SELECT * FROM users WHERE email = '" + email + "'"`.
- **NoSQL injection**: `db.users.find({email: req.body.email})` when the body sends `{$gt: ""}`.
- **OS command injection**: `exec("convert " + filename + ".jpg")`.
- **LDAP, XPath, ORM, GraphQL injection** — same pattern, different language.
- **Cross-site scripting (XSS)** — injection of client-side JavaScript into HTML the browser renders.
- **Template injection** — user data interpreted as template expressions (`{{ 7*7 }}` returns `49`).
- **CRLF injection** — newlines into headers (response splitting).

**Mitigations:**
- **Parameterized queries / prepared statements.** Never string-concatenate SQL.
- **ORMs used correctly** — many still escape if you go off-spec.
- **Strict input validation** — typed schemas, allowlists.
- **Output encoding** — context-aware HTML escaping for XSS.
- **Content Security Policy (CSP)** — strong defense-in-depth against XSS.
- For shell exec: avoid; if unavoidable, use array-form `execve()`-style APIs that don't invoke a shell.

```python
# WRONG
cur.execute(f"SELECT * FROM users WHERE id = {user_id}")

# RIGHT
cur.execute("SELECT * FROM users WHERE id = %s", (user_id,))
```

---

## 5. A04 — Insecure Design

**What it means:** flaws baked into how the system is designed, not just how it's coded. A correct implementation of an insecure design is still insecure.

**Manifestations:**
- "Account recovery via security questions" (mother's maiden name → easily looked up).
- "Forgot password sends new password by email."
- A bank app with no transaction limits or out-of-band confirmation on transfers.
- A signup flow that accepts unlimited account creation per IP.
- Trust-implication: "the gateway already validated, so internal service doesn't need to."
- A scheduling system that exposes calendar links via guessable URLs.

**Mitigations:**
- **Threat modeling.** STRIDE, attack trees, abuse-case stories. Done early, repeated as design changes.
- **Secure design patterns**: rate-limited resources, reversible operations, idempotent APIs, principle of least privilege, defense-in-depth.
- **Misuse cases in user stories**, not just happy paths.
- **Security design reviews** before building.
- Reference: OWASP SAMM, NIST SSDF.

---

## 6. A05 — Security Misconfiguration

**What it means:** the platform, framework, or app has settings that leak data or expand attack surface. Includes default passwords, debug mode in production, verbose error pages, missing headers, exposed admin interfaces.

**Manifestations:**
- Cloud storage buckets publicly readable.
- Database accessible from the internet (default `bind 0.0.0.0`).
- Stack traces returned to users.
- Default admin credentials still active.
- Unused features enabled (XML-RPC, sample apps, debug endpoints).
- Missing HTTP security headers (`X-Content-Type-Options`, `Strict-Transport-Security`, `Content-Security-Policy`).
- Overly permissive CORS (`Access-Control-Allow-Origin: *` with credentials).

**Mitigations:**
- Hardened baseline images / configurations.
- Infrastructure as Code (Terraform, Pulumi) with security scans (Checkov, tfsec).
- Continuous configuration scanning (AWS Config, Forseti, Wiz, Prowler).
- Generic error messages in production.
- Headers via reverse proxy or middleware:

```
Strict-Transport-Security: max-age=63072000; includeSubDomains
Content-Security-Policy: default-src 'self'; ...
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
Referrer-Policy: strict-origin-when-cross-origin
Permissions-Policy: geolocation=(), microphone=()
```

---

## 7. A06 — Vulnerable & Outdated Components

**What it means:** you ship a library with a known CVE. Or a base container image. Or a kernel.

**Manifestations:**
- Log4j 2.x in a Java service (Log4Shell, CVE-2021-44228).
- Spring 4 with deserialization holes.
- npm package with a known prototype-pollution CVE.
- Outdated nginx without ALPACA patch.
- Old K8s with kubectl CVE.
- Forgotten subdomain pointing to abandoned S3 bucket → subdomain takeover.

**Mitigations:**
- **SBOM** (Software Bill of Materials) for every artifact you ship.
- **Dependency scanning** (Dependabot, Renovate, Snyk, GitHub Advanced Security, AWS Inspector).
- **Container scanning** (Trivy, Grype, Sysdig).
- **Patching cadence**: critical CVE within days, high within weeks.
- **Reduce dependencies.** Each one is liability.
- **Update or retire** unmaintained dependencies.

---

## 8. A07 — Identification & Authentication Failures

**What it means:** the login system is wrong. Credential stuffing works. Sessions never expire. MFA is bypassable. Tokens are predictable.

**Manifestations:**
- No rate limit on `/login`; attackers brute force.
- Weak session IDs.
- Session not invalidated on logout.
- Password reset tokens valid for 7 days.
- "Remember me" cookies that never expire.
- No MFA.
- MFA implemented only on login, not on sensitive actions (step-up).
- Username enumeration via differential error messages.
- JWT `alg=none` accepted.

**Mitigations:**
- Strong password hashing (Argon2id / bcrypt).
- Per-account + per-IP rate limiting with progressive backoff.
- MFA, prefer phishing-resistant (WebAuthn / passkeys).
- Short session lifetimes, idle timeouts, server-side revocation.
- Same response time for "user not found" and "wrong password."
- Single-use, short-lived reset tokens.
- See [Authentication vs Authorization →](./authn-vs-authz.md), [Session-Based Authentication →](./sessions.md), [Password Storage →](./password-storage.md).

---

## 9. A08 — Software & Data Integrity Failures

**What it means:** code or data is trusted without verification. Supply chain attacks, insecure deserialization, auto-update without signature checks.

**Manifestations:**
- Java/PHP/Python apps deserializing untrusted data → RCE (Log4Shell's cousin).
- CI pipelines pulling unsigned dependencies and shipping them.
- A `curl | bash` install script with no checksum or signature.
- Auto-update mechanism without code signing.
- Container image pulled by tag (mutable) rather than digest.
- Build artifacts not signed (no SLSA provenance).
- "We just trust our private npm registry" — until an internal package is replaced by a malicious one (dependency confusion, 2021).

**Mitigations:**
- **Code signing** for releases.
- **Signed container images** (Sigstore, cosign, Notary).
- **Reproducible builds** where feasible.
- **SLSA framework** levels for build pipelines.
- **Lockfiles + integrity hashes** (`package-lock.json` with `integrity`, `Cargo.lock`, `poetry.lock`).
- **No untrusted deserialization** — use JSON, avoid Java native serialization, Python pickle for cross-trust boundaries.
- **Pin to digests, not tags**, for container images and dependencies.

---

## 10. A09 — Security Logging & Monitoring Failures

**What it means:** breaches go undetected because nobody logged or monitored the right things. Industry average to detect a breach is months.

**Manifestations:**
- No logs of failed login attempts.
- Logs that flood and are never searched.
- No alerting on anomalies (50× failed logins, password resets, admin actions).
- Logs deleted within days; can't reconstruct an incident.
- Sensitive data (passwords, tokens) ending up in logs.
- No central log aggregation; each service logs locally and disappears with the pod.
- No tamper-evident audit log for security-sensitive operations.

**Mitigations:**
- Log: logins (success/fail), high-value actions (role changes, password resets, exports), authorization denials, system errors.
- **Centralize logs** (ELK, Loki, Splunk, Datadog).
- **Structured logging** with correlation IDs.
- **Alerting on signals**: rate-of-failure spikes, geo anomalies, off-hours admin activity.
- **Immutable audit trail** for sensitive ops (append-only, separate storage).
- **Retention** at least 90 days, often 1+ year for compliance.
- See [Logging Best Practices →](../13-observability/logging.md) and [Alerting & On-Call →](../13-observability/alerting.md).

---

## 11. A10 — Server-Side Request Forgery (SSRF)

**What it means:** the server fetches a URL the attacker controls — and the attacker uses that to reach things the attacker couldn't reach directly.

**Why it's nasty:** in cloud environments, the metadata service (`http://169.254.169.254/`) hands out IAM credentials. SSRF that hits the metadata service = cloud account compromise. This is exactly how Capital One was breached in 2019.

**Manifestations:**
- An image-uploader that accepts a URL and fetches it server-side.
- A webhook receiver that calls back to a customer URL.
- A PDF renderer that resolves `<img src>` server-side.
- A "preview link" feature.
- An OpenID connect callback that fetches the issuer config from a user-supplied URL.

**Mitigations:**
- **Disallow internal IP ranges**: 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16, 127.0.0.0/8, 169.254.0.0/16 (metadata), ::1, fc00::/7.
- **Resolve and check IPs** after DNS resolution (DNS rebinding-aware). Verify the resolved IP is allowed *before* connecting.
- **Allowlist of permitted destinations** rather than denylist.
- **Disable redirects, or follow with the same checks each hop.**
- **Use IMDSv2** on AWS — requires session token, defeats most SSRF-to-metadata attacks.
- **Egress filter** at the network layer.
- **Network segmentation** so even successful SSRF can't reach sensitive backends.

---

## 12. The OWASP API Security Top 10 — A Quick Note

For API-heavy services, also know the API list (last updated 2023):

1. **Broken Object Level Authorization (BOLA)** — IDOR, the #1 API risk.
2. **Broken Authentication**.
3. **Broken Object Property Level Authorization** — exposing fields you shouldn't (mass-assignment).
4. **Unrestricted Resource Consumption** — no rate limits, large payloads.
5. **Broken Function Level Authorization** — admin endpoints accessible by anyone.
6. **Unrestricted Access to Sensitive Business Flows** — bots buying every concert ticket.
7. **SSRF**.
8. **Security Misconfiguration**.
9. **Improper Inventory Management** — undocumented APIs.
10. **Unsafe Consumption of APIs** — trusting third-party API responses too much.

For SaaS APIs, OWASP API Top 10 is the more relevant list day-to-day.

---

## 13. How to Use the Top 10

It's not a checklist; it's a curriculum. Approaches:

- **Training.** Every engineer should be able to name and describe each item.
- **Code review heuristics.** "Where could broken access control hide here?" applied to a diff.
- **Threat modeling input.** Each Top 10 item becomes an attacker capability to consider.
- **Pen-test scoping.** "Test for the OWASP Top 10" is a standard contract clause.
- **Compliance.** PCI DSS, SOC 2, and others reference it.

The trap: treating it as exhaustive. The Top 10 is the floor, not the ceiling.

---

## 14. Common Mistakes / Anti-Patterns (Meta)

- **Reading the Top 10 once and forgetting.** Reread when designing or reviewing.
- **Equating "no Top 10 findings" with "secure."** Many breaches are subtle business-logic flaws not on this list.
- **Skipping the official long descriptions.** OWASP's writeups include real CWEs, examples, and references — the one-line summaries lose nuance.
- **Targeting only the perimeter.** Most of these manifest deep in app logic.
- **Treating Top 10 as a substitute for threat modeling.** Top 10 is generic; threat modeling is specific to your system.
- **Top 10 fatigue: "We solved A03 in 2020, why are you still asking?"** New code resurrects old issues.

---

## 15. Cheat Card

```
OWASP TOP 10 (2021 — current)

A01  Broken Access Control       authorize per RESOURCE, central policy, default deny
A02  Cryptographic Failures      HTTPS, AES-GCM, Argon2id, KMS, no DIY crypto
A03  Injection                   parameterized queries, output encoding, CSP
A04  Insecure Design             threat modeling, misuse cases, design review
A05  Security Misconfiguration   IaC + scanners, headers, no debug in prod
A06  Vulnerable Components       SBOM, Dependabot, scan + patch fast
A07  AuthN Failures              rate limit, MFA, strong hashing, short sessions
A08  Integrity Failures          sign code/images, SLSA, no untrusted deserial
A09  Logging/Monitoring          centralize, alert, immutable audit, retain ≥ 90 d
A10  SSRF                        block private IPs, allowlist destinations, IMDSv2

API TOP 10: BOLA is #1. Always.

USE     curriculum + code-review lens + threat-model input
DON'T   treat as exhaustive · check once and forget · ignore for "internal" code

RULE: every diff should be examined for at least one Top 10 risk it touches.
```

---

## 16. Resources

### Documentation
- **OWASP Top 10 (2021)** — <https://owasp.org/Top10/>
- **OWASP API Security Top 10 (2023)** — <https://owasp.org/API-Security/>
- **OWASP Cheat Sheet Series** — <https://cheatsheetseries.owasp.org/>
- **OWASP ASVS** (Application Security Verification Standard) — checklist-grade requirements.
- **NIST SSDF (Secure Software Development Framework)** — <https://csrc.nist.gov/projects/ssdf>
- **CWE Top 25** — <https://cwe.mitre.org/top25/>

### Articles
- "How Capital One was breached" — SSRF + IMDSv1 case study.
- "Lessons from the Equifax breach" — A06 in vivid detail.
- Krebs on Security, The Daily Swig — incident roundups mapping to Top 10.

### Books
- *The Web Application Hacker's Handbook* — Stuttard & Pinto. Still the classic.
- *Real-World Bug Hunting* — Peter Yaworski. Practical examples for each category.
- *Web Application Security* — Andrew Hoffman.
- *Alice and Bob Learn Application Security* — Tanya Janca.

### Videos
- OWASP YouTube channel — annual conferences, talks per category.
- LiveOverflow — vulnerability deep dives.
- PortSwigger Web Security Academy — free, hands-on labs per Top 10 category.

### Tools
- **OWASP ZAP, Burp Suite** — interactive scanners.
- **Snyk, Dependabot, Renovate, Trivy** — dependency / image scanning.
- **Semgrep, CodeQL** — static analysis.
- **Nuclei** — templated vulnerability scanner.
- **Gitleaks, TruffleHog** — secrets scanning.
- **AWS Inspector, Wiz, Lacework, Prowler** — cloud security posture.

### Adjacent reading
- [Authentication vs Authorization →](./authn-vs-authz.md)
- [RBAC, ABAC, ReBAC →](./access-control.md)
- [Encryption at Rest & In Transit →](./encryption.md)
- [Hashing, Salting, Password Storage →](./password-storage.md)
- [JWT — JSON Web Tokens →](./jwt.md)
- [DDoS Protection & WAF →](./ddos-waf.md)
- [Zero Trust Architecture →](./zero-trust.md)
- [Secrets Management →](./secrets-management.md)
- [Logging Best Practices →](../13-observability/logging.md)

---

*Previous:* [← Zero Trust Architecture](./zero-trust.md)  |  *Up:* [README ↑](../README.md)
