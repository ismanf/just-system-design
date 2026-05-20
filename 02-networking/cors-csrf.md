# CORS, CSRF, Same-Origin Policy

> **TL;DR** — Browsers have a built-in security policy called the **Same-Origin Policy (SOP)** that prevents JavaScript on one site from reading data on another. **CORS** (Cross-Origin Resource Sharing) is the *server's* way of opting in to specific cross-origin reads. **CSRF** (Cross-Site Request Forgery) is a separate attack where a malicious site makes a *request* with your cookies attached, abusing the trust the target site places in the browser; you mitigate it with **CSRF tokens**, **SameSite cookies**, and good auth design. These three together (SOP / CORS / CSRF) define how cross-site web security actually works in 2026.

---

## 1. Origin — The Atomic Unit of Web Security

An **origin** = `scheme + host + port`. All three must match.

```
https://example.com:443    ─┐
https://example.com         ─┘  same origin

http://example.com           ─┐
https://example.com           ─┘  different (scheme)

https://example.com           ─┐
https://api.example.com       ─┘  different (host)

https://example.com:443       ─┐
https://example.com:8443      ─┘  different (port)
```

Subdomains are *not* the same origin. `app.example.com` and `api.example.com` are cross-origin even though they share a parent domain.

---

## 2. The Same-Origin Policy (SOP)

By default, JavaScript running on `https://a.com` **cannot read** responses from `https://b.com`. SOP applies to:

- `fetch` / `XMLHttpRequest` responses.
- Reading the contents of an `<iframe>` from another origin.
- Reading pixel data from `<canvas>` tainted by cross-origin images.
- Reading `localStorage` / `sessionStorage` / cookies from another origin.

What SOP **does not** block:
- *Sending* a cross-origin request (the request goes; the response is hidden).
- `<img>`, `<script>`, `<link>` tag loads (cross-origin GETs without reading the body).
- Form submissions to another origin.
- Top-level navigations.

This asymmetry is why CSRF exists: the request still goes through (cookies included), even though the response is hidden.

---

## 3. CORS — Opt-In Cross-Origin Reads

A cross-origin request still happens; the browser just refuses to give the response to JS — *unless* the server explicitly opts in with CORS headers.

```
Browser tab on https://app.com
   │
   │ fetch("https://api.example.com/me")
   ▼
api.example.com replies with:
   Access-Control-Allow-Origin: https://app.com
   ↓
Browser hands the response to app.com's JS. ✅
```

Without the right header? The fetch resolves but JS sees an opaque error and no body.

### Simple requests vs preflighted requests

Some cross-origin requests are "simple" — the browser sends them directly. Others require a **preflight** `OPTIONS` request to ask "may I?" first.

**Simple** request criteria (all must hold):
- Method is `GET`, `HEAD`, or `POST`.
- Content-Type is one of: `application/x-www-form-urlencoded`, `multipart/form-data`, or `text/plain`.
- No custom headers (only a small allowlist of "CORS-safelisted" headers).
- No `ReadableStream` body.

Everything else → **preflight**.

### A preflight in action
```
Browser:                                Server:
OPTIONS /users HTTP/1.1                 HTTP/1.1 204 No Content
Origin: https://app.com                 Access-Control-Allow-Origin: https://app.com
Access-Control-Request-Method: PUT      Access-Control-Allow-Methods: GET, PUT, POST
Access-Control-Request-Headers:         Access-Control-Allow-Headers: content-type,
   content-type, authorization                                       authorization
                                        Access-Control-Max-Age: 600

(if approved, then the real request:)
PUT /users HTTP/1.1                     HTTP/1.1 200 OK
Origin: https://app.com                 Access-Control-Allow-Origin: https://app.com
Content-Type: application/json          {...}
Authorization: Bearer ...

{"name":"Ada"}
```

The `Access-Control-Max-Age` lets the browser cache the preflight result so subsequent calls skip the extra round trip.

### The CORS response headers you'll use

| Header | Purpose |
| --- | --- |
| `Access-Control-Allow-Origin` | Which origin(s) may read. `*` allows any (but disables credentials). |
| `Access-Control-Allow-Methods` | Allowed HTTP methods on preflight. |
| `Access-Control-Allow-Headers` | Allowed request headers on preflight. |
| `Access-Control-Allow-Credentials` | If `true`, cookies / Authorization are sent. Requires explicit origin, not `*`. |
| `Access-Control-Expose-Headers` | Response headers JS may read (default: only CORS-safelisted). |
| `Access-Control-Max-Age` | Seconds to cache the preflight. |

### Credentials and CORS

When `fetch(url, {credentials: "include"})` is used, the browser sends cookies. The server *must* respond with:
- `Access-Control-Allow-Credentials: true`
- A specific `Access-Control-Allow-Origin` (NOT `*`).

If you use `*` with credentials, browsers refuse.

### Common server configuration

Most frameworks have a CORS middleware. The right defaults:

```
Access-Control-Allow-Origin: https://app.example.com     ← specific origin, NEVER * for credentialed APIs
Access-Control-Allow-Credentials: true
Access-Control-Allow-Methods: GET, POST, PUT, PATCH, DELETE
Access-Control-Allow-Headers: Content-Type, Authorization, X-Request-Id
Access-Control-Max-Age: 600
Vary: Origin
```

`Vary: Origin` is critical when you reflect the origin dynamically — caches need to know responses differ per origin.

### Reflecting the origin (be careful)
A common pattern: maintain a list of allowed origins, and if the request's `Origin` header matches, echo it back. Done wrong, this becomes "allow *any* origin", which is dangerous. Always **validate** against a precise allowlist; never blindly echo.

---

## 4. CORS vs Same-Origin: Common Confusions

- **CORS does NOT make your API more secure.** It controls which *browser scripts* can read your responses. Non-browser clients (curl, mobile, server-to-server) ignore CORS entirely.
- **CORS is enforced by the browser, not the network.** Curl sees everything; the browser hides it.
- **Disabling CORS is not "fixing" anything.** It's removing the *browser*'s opt-in protection.
- **`Access-Control-Allow-Origin: *`** doesn't allow credentials.
- **Cookies for cross-site requests** also depend on **`SameSite`** cookie attribute (next section). CORS alone isn't enough.

---

## 5. CSRF — Cross-Site Request Forgery

### The attack
You're logged into `bank.com`. The site stores a session cookie. While still logged in, you visit `evil.com`, which contains:

```html
<form action="https://bank.com/transfer" method="POST">
  <input name="to" value="attacker" />
  <input name="amount" value="1000" />
</form>
<script>document.forms[0].submit()</script>
```

The browser dutifully sends the form to `bank.com` — and the browser **attaches your bank.com cookies**. Bank sees a properly authenticated request and transfers the money. SOP didn't help: the *request* was allowed; the *response* would be hidden, but the side effect is done.

That's CSRF: tricking a victim's browser into making a state-changing request, abusing the cookies it carries.

### What enables CSRF
- Auth via **cookies** that the browser auto-attaches.
- State-changing endpoints that act on a *single* request without proof of intent.
- The victim is logged in.

### Defenses (use more than one)

#### 1. SameSite cookies (the modern default)
Set your auth cookie:
```
Set-Cookie: session=abc; SameSite=Lax; Secure; HttpOnly
```
- `SameSite=Lax` (modern browser default) — cookie *not* sent on most cross-site requests, except top-level GET navigations.
- `SameSite=Strict` — never sent on cross-site requests.
- `SameSite=None; Secure` — sent everywhere (required for embedded SaaS, payment flows, etc.).

This alone blocks most classic CSRF. But always combine with:

#### 2. CSRF tokens (synchronizer pattern)
- Server issues a random token tied to the session, e.g. in a hidden form field or a custom header.
- Client must include it on every state-changing request.
- Server rejects requests missing or with the wrong token.

Attacker on `evil.com` can submit a form but **can't read** the token (SOP blocks reading bank.com's HTML response → no token). The forged request fails.

#### 3. Double-submit cookies
Server sets a random value as a cookie; client must also send it back as a header. The attacker can't read or set that cookie → can't match the header.

#### 4. Custom header on JSON APIs
A POST that requires `Content-Type: application/json` and a custom header like `X-Requested-With: XMLHttpRequest` is **not a simple request** → triggers CORS preflight. The browser won't send the preflight successfully unless the origin is allowed → attacker fails.

#### 5. Re-authenticate sensitive actions
Money transfers, password changes — require the password again. CSRF can't fill that in.

#### 6. Use `Authorization: Bearer <token>` instead of cookies
Tokens stored in JS (e.g., in memory) are not automatically sent by the browser. The attacker can't make a cross-origin request with your token attached. (But now you've shifted some risk to XSS — guard against that separately.)

### CSRF vs XSS
- **CSRF**: attacker tricks the *browser* into sending a request with the victim's credentials. The attacker never sees the response.
- **XSS**: attacker injects script that runs *on your origin*, with full access to cookies, tokens, DOM.

XSS is strictly worse. XSS protections (CSP, output encoding, sanitization) sit alongside CSRF protections.

---

## 6. SameSite Cookies — The Modern Default

Browsers (since 2020) default cookies to `SameSite=Lax`. That single change blocked the cheapest variety of CSRF (cross-site POST from a third-party form).

Quick reference:

| Mode | Sent on same-site? | Sent on cross-site top-level GET? | Sent on cross-site iframe / fetch? |
| --- | --- | --- | --- |
| `Strict` | ✅ | ❌ | ❌ |
| `Lax` (default) | ✅ | ✅ (only `GET`/`HEAD`/safe) | ❌ |
| `None` (must be `Secure`) | ✅ | ✅ | ✅ |

Rule of thumb:
- **First-party app cookie** → `SameSite=Lax; Secure; HttpOnly`.
- **Cross-site embed** (SaaS widget, OAuth pop-up, payment iframe) → `SameSite=None; Secure`.
- **Strict** is rarely used outside ultra-sensitive flows; it breaks "click email link, land on logged-in dashboard".

---

## 7. CORS + CSRF — How They Interact

People conflate them. They're not the same.

- **CORS** controls what *responses* JS can *read* across origins.
- **CSRF** controls what *requests* the server should *trust* across origins.

A wide-open CORS policy *can* enable CSRF if you also rely on cookies — because now the attacker's JS can read the response too. Tight CORS + SameSite cookies + CSRF tokens is the belt-and-suspenders default.

---

## 8. Cookies, Tokens, and the Modern Web App

In 2026, three common auth styles:

### A — Cookies (first-party SPA on same domain)
- Server sets `HttpOnly; Secure; SameSite=Lax` cookie.
- Browser auto-attaches.
- CSRF mitigated by SameSite + CSRF token.
- Best for traditional web apps (SSR, Rails-style, Django).

### B — Bearer tokens in memory (third-party SPA / cross-origin API)
- Login returns a JWT or opaque token.
- Stored in memory (or short-lived storage); attached as `Authorization: Bearer ...`.
- CSRF basically impossible (no auto-attach).
- XSS risk: anything that can run JS can read the token — strong CSP required.

### C — OAuth / OIDC with refresh-token cookie
- Short-lived access token in memory.
- Refresh token in an `HttpOnly; Secure; SameSite=Strict` cookie scoped to `/auth`.
- Combines low XSS risk (access token short-lived) with CSRF safety (refresh cookie is Strict).

Most SaaS today is some flavor of B or C.

---

## 9. Worked Example: Configuring a Cross-Origin API

You're running:
- `https://app.example.com` (SPA)
- `https://api.example.com` (API)

In your API:
```http
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Credentials: true
Access-Control-Allow-Methods: GET, POST, PUT, PATCH, DELETE
Access-Control-Allow-Headers: Content-Type, Authorization, X-CSRF-Token
Access-Control-Max-Age: 600
Vary: Origin
```

Cookies:
```http
Set-Cookie: session=abc; Domain=.example.com; Path=/; HttpOnly; Secure; SameSite=Lax
```

Frontend fetch:
```js
fetch("https://api.example.com/me", {
  credentials: "include",
  headers: { "X-CSRF-Token": csrfToken },
});
```

CSRF token comes from a `GET /csrf` endpoint at app load, stored in JS memory, sent on every mutating call. Server verifies it.

That's the canonical setup.

---

## 10. Common Mistakes

### CORS
- `Access-Control-Allow-Origin: *` with credentials — refused by the browser, but people still try it.
- Reflecting the `Origin` header without an allowlist — opens up the API to any site.
- Forgetting `Vary: Origin` → cached cross-origin responses pollute other users.
- Adding CORS to "fix" 401 errors — CORS isn't about *auth*.
- Allowing methods you don't need (e.g., `DELETE` on a read-only API).
- Forgetting `OPTIONS` in your firewall / WAF rules.
- Treating CORS as a server-side firewall. *Curl ignores it.*

### CSRF
- Relying on `Origin` / `Referer` only — those headers are sometimes stripped.
- No CSRF token *and* no SameSite=Lax/Strict cookies.
- CSRF token tied only to a *cookie* (the attacker can also send the cookie) — token must be bound to the *session* and validated server-side.
- Disabling CSRF protection because "we have CORS" — CORS doesn't stop cross-site requests.
- Long-lived bearer tokens stored in `localStorage` — XSS reads them, far worse than CSRF.

---

## 11. Debugging CORS Issues

The error in DevTools usually reads:
> *Access to fetch at 'https://api.example.com/...' from origin 'https://app.example.com' has been blocked by CORS policy: ...*

Diagnostic order:
1. **Is the request reaching the server?** Inspect Network tab → response status. The browser *blocks the JS from reading* but the request often *did go through*.
2. **Is the preflight succeeding?** Look for an `OPTIONS` line. Inspect its response headers.
3. **Does `Access-Control-Allow-Origin` match the requesting origin exactly?** Spelling, scheme (http vs https), trailing slashes.
4. **If credentials are needed**, is `Access-Control-Allow-Credentials: true` present *and* the origin not `*`?
5. **Custom headers used?** They must be listed in `Access-Control-Allow-Headers`.
6. **Server returning a redirect on the preflight?** Browsers don't follow redirects on preflight.
7. **Proxy / WAF stripping headers?** Some corp proxies drop unfamiliar headers.

`curl` tests:
```bash
# Simulate a preflight
curl -i -X OPTIONS https://api.example.com/users \
  -H "Origin: https://app.example.com" \
  -H "Access-Control-Request-Method: POST" \
  -H "Access-Control-Request-Headers: content-type"
```

---

## 12. Beyond SOP/CORS/CSRF — Adjacent Browser Security

You'll meet these in the same conversations:

- **Content Security Policy (CSP)** — restricts what scripts/styles your page can load. Best XSS mitigation.
- **Subresource Integrity (SRI)** — hashes for `<script>` / `<link>` tags so CDN tampering is detected.
- **Cross-Origin-Opener-Policy (COOP)**, **Cross-Origin-Embedder-Policy (COEP)**, **Cross-Origin-Resource-Policy (CORP)** — newer headers; needed for `SharedArrayBuffer` and Spectre mitigations.
- **Referrer-Policy** — controls what `Referer` header is leaked cross-site.
- **Strict-Transport-Security (HSTS)** — forces HTTPS.
- **X-Frame-Options / `frame-ancestors`** — prevents clickjacking by disallowing your site in iframes.

A well-configured site enables most of these via headers from the reverse proxy or framework.

---

## 13. Cheat Card

```
ORIGIN          = scheme + host + port. All three must match for "same".
SOP             = browsers block reading cross-origin responses by default.
CORS            = the SERVER's opt-in to allow cross-origin READS.
  - simple req: GET/HEAD/POST + safe Content-Type + no custom headers
  - preflight OPTIONS otherwise
  - credentials needs specific Origin (NOT *), and Allow-Credentials: true.
  - Vary: Origin when reflecting.

CSRF            = attacker tricks YOUR BROWSER into sending a request
                  that carries YOUR cookies. SOP doesn't help (the request goes).
  Defenses (combine):
    1. SameSite=Lax/Strict cookies (modern default).
    2. CSRF tokens (synchronizer pattern).
    3. Custom header that triggers preflight.
    4. Re-auth for sensitive actions.
    5. Bearer tokens (not cookies) — but watch XSS.

NOT THE SAME
  CORS protects browser-script reads.
  CSRF protects server-trust of requests.
  XSS = arbitrary JS on your origin = catastrophic; CSP + sanitization.

REMEMBER
  curl / mobile / server-to-server clients IGNORE CORS.
  CORS isn't auth; it's a browser-only opt-in.
```

---

## 14. Resources

### Specs
- **Fetch standard / CORS**: <https://fetch.spec.whatwg.org/>
- **Same-Origin Policy** (MDN): <https://developer.mozilla.org/en-US/docs/Web/Security/Same-origin_policy>
- **CSRF** (MDN): <https://developer.mozilla.org/en-US/docs/Web/Security/Types_of_attacks#cross-site_request_forgery_csrf>
- **SameSite cookies** spec: <https://datatracker.ietf.org/doc/html/draft-ietf-httpbis-rfc6265bis>

### Guides
- MDN — CORS: <https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS>
- OWASP — CSRF Prevention Cheat Sheet: <https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html>
- OWASP — SameSite cookies: <https://cheatsheetseries.owasp.org/cheatsheets/SameSite_Cheatsheet.html>
- Web.dev SameSite cookies: <https://web.dev/articles/samesite-cookies-explained>

### Books
- *The Tangled Web* — Michal Zalewski. Pre-CORS but the canonical book on browser security.
- *Web Application Security* — Andrew Hoffman (O'Reilly).

### Videos
- ByteByteGo: "What is CORS?" — <https://www.youtube.com/@ByteByteGo>
- ByteByteGo: "CSRF Explained" — same channel.
- Hussein Nasser CORS/CSRF deep dives — <https://www.youtube.com/@hnasr>
- Computerphile on browser security topics.

### Adjacent reading
- [HTTPS, TLS/SSL Handshake](./https-tls.md)
- [OAuth 2.0 & OpenID Connect →](../12-security/oauth-oidc.md)
- [JWT — JSON Web Tokens →](../12-security/jwt.md)
- [OWASP Top 10 →](../12-security/owasp-top-10.md)

---

*Previous:* [← Webhooks](./webhooks.md)  |  *Up:* [README ↑](../README.md)
