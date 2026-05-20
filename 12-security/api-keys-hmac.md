# API Keys, HMAC Signing

> **TL;DR** — **API keys** are long-lived secret strings used by clients (usually other services) to authenticate to your API. They're simple, ubiquitous, and dangerous when leaked. **HMAC request signing** adds integrity: the client signs each request with a shared secret, and the server verifies the signature. HMAC defeats replay and tampering even when the wire is intercepted, which is why AWS, Stripe webhooks, GitHub webhooks, and Shopify all sign requests. Bearer keys are fine for backend-to-backend with TLS. Signed requests are required when you can't trust the channel, when you need replay protection, or when you're sending the request from the user's browser.

---

## 1. API Keys — The Simplest Credential

An API key is just a string the server has previously issued and remembers:

```
Authorization: Bearer ***
```

Or sometimes:

```
X-API-Key: ***
```

The server looks it up in a database, finds the associated principal (account, project, scopes), and authorizes the request. Done. This is how Stripe's secret keys, OpenAI keys, SendGrid keys, Twilio Auth Tokens, and a thousand other APIs work.

### What an API key is

- **Long-lived** — created once, used until revoked. Typical lifetime: months to years.
- **Static** — the same string is sent on every request.
- **Bearer** — anyone holding it can use it (no possession proof).
- **Scope-bound** — should be tied to a set of permissions (read-only, account-restricted, etc.).
- **Prefixed** — modern keys have prefixes like `sk_live_`, `gho_`, `xoxb-` so they're greppable in source-code leaks and identifiable by scanners.

### What an API key is **not**

- Not user authentication — it doesn't identify *which human* is using it. Tying actions to humans requires another layer (e.g., audit logs tracking which user generated the key).
- Not a password — it's longer, machine-generated, and never user-typed.
- Not a token (in the OAuth sense) — there's no expiry, no issuer, no signature.

---

## 2. Where API Keys Belong

Use API keys when:
- **Server-to-server** calls inside trusted networks or over TLS to a known third party.
- **Developer integrations** where humans paste a key into config files (Stripe, OpenAI, Twilio, Datadog).
- **Webhook authentication** — the receiver gives the sender a key to include.
- **CI/CD pipelines** — short-lived keys preferred, but long-lived ones are common.

**Don't** use API keys when:
- The credential must travel through the user's browser. API keys + bearer auth + browser = the key is now in the user's network, the browser console, the LocalStorage, and possibly the page DOM.
- The action must be tied to a human user.
- Instant revocation must propagate across a federated identity ecosystem (use OAuth).
- The request goes over an untrusted channel where replay is a risk (use HMAC signing).

---

## 3. Generating and Storing Keys Properly

### Generation

```python
import secrets
key = "sk_live_" + secrets.token_urlsafe(32)   # 256 bits of entropy
```

128 bits minimum. Use a CSPRNG. Don't use UUIDs (they look random but reveal version + timestamp info and are sometimes guessable).

### Storage at the server

You'd think you store the key in a database. **Don't** — at least not in plaintext. If your DB leaks, every key leaks.

Two patterns:

**Hash the key on storage** (like a password):
```
DB row: { id, prefix: "sk_live_4eC39H", hash: bcrypt(full_key), ... }
Verify: bcrypt(submitted) == stored_hash
```

- Pros: DB leak doesn't reveal keys.
- Cons: bcrypt per request is slow. Cache verified keys in memory with short TTL.

**Store encrypted, decrypt in memory** (with KMS):
```
DB row: { id, prefix, encrypted_key (AES-GCM via KMS) }
```

- Pros: Fast verification (decrypt + compare).
- Cons: KMS access = key access. Compromised app server = leaked keys.

Stripe's approach (publicly documented): key prefix stored plaintext for lookup, key body hashed. Lookups use the prefix; verification compares hashes.

### Display once

When a key is created, show it to the user **exactly once**. After that, only show the prefix. If they lose it, they rotate it. Every modern API console works this way.

### Prefixes (do them)

`sk_live_` (Stripe live secret), `sk_test_` (Stripe test), `gho_` (GitHub OAuth token), `glpat-` (GitLab personal access token), `xoxb-` (Slack bot). Prefixes:
- Help scanners (GitHub secret scanning, TruffleHog) detect leaked keys.
- Make logs greppable.
- Distinguish key types (live/test/restricted).

---

## 4. Scoping and Restricting Keys

A leaked key with full account access is a disaster. A leaked key restricted to "read invoices for project X from IP 1.2.3.4" is annoying but contained.

Restrict by:

| Dimension | Example |
| --- | --- |
| **Scope / permissions** | `read:users`, `write:orders` |
| **Account / project** | Tied to one tenant in a multi-tenant system |
| **IP allowlist** | Only valid from 10.0.0.0/8 |
| **Origin allowlist** (for browser-side public keys) | Only from `https://app.example.com` |
| **Rate limit** | Lower limit on more-permissive keys |
| **Expiry** | Most should expire — even "long-lived" keys benefit from 90-day rotation |

**Modern best practice:** keys are restricted by default; users opt into permissions explicitly. Stripe's restricted keys are the model.

---

## 5. Rotation

Every API key must have a rotation story. The standard pattern:

```
1. Create key v2 alongside key v1.
2. Update consumers to use v2.
3. Once 0% of traffic uses v1, delete v1.
```

Two keys must be valid simultaneously during overlap. This means the system supports **multiple active keys per principal**, not "one key per account."

Default rotation cadence: 90 days for human-issued keys, 30 days for automation-generated keys. After an incident, rotate immediately.

Tooling: AWS Secrets Manager, HashiCorp Vault, GCP Secret Manager all support rotation hooks. See [Secrets Management →](./secrets-management.md).

---

## 6. HMAC Request Signing — Why and How

API keys plus HTTPS are usually enough. But there are cases where they aren't:

1. **Webhooks.** Your server sends a callback to a customer's URL. You don't trust their TLS termination — the request crosses multiple proxies, may be replayed, may be tampered with. The receiver needs to verify the payload came from you.
2. **Cloud APIs at scale.** AWS, GCS, Cloudflare R2 use signed requests so that a leaked key isn't usable without the signing process (which they audit), and so that requests can be presigned with bounded validity (S3 presigned URLs).
3. **Browser-originated requests** where you can't safely embed a secret in the page but can sign requests through a trusted proxy.

The mechanism: HMAC (Hash-based Message Authentication Code).

```
signature = HMAC_SHA256(secret, message)
```

The client computes the signature, sends it alongside the request. The server recomputes it using the same shared secret and compares. If they match, the message came from someone who knows the secret and was not modified in transit.

### What "message" means

You don't sign random bytes — you sign a canonical string built from the request:

```
StringToSign = HTTP_METHOD + "\n" +
               REQUEST_PATH + "\n" +
               SORTED_QUERY_STRING + "\n" +
               SELECTED_HEADERS + "\n" +
               SHA256(BODY) + "\n" +
               TIMESTAMP
```

The exact format is per-API. AWS Signature Version 4, Stripe, GitHub webhooks, Shopify, Slack — all slightly different. Always read the spec.

```
Authorization: Signature
    keyId="acct_123",
    algorithm="hmac-sha256",
    headers="(request-target) host date",
    signature="base64(HMAC(secret, signing_string))"
```

---

## 7. Replay Protection

A signature alone doesn't stop replay: an attacker captures a valid signed request and re-sends it.

Standard mitigations:

- **Timestamp** in the signature, server rejects if older than N minutes (Stripe: 5 minutes). Requires reasonable clock sync.
- **Nonce** included and tracked: server stores recent nonces, rejects duplicates.
- **One-time URLs**: presigned with `expires_at`, no nonce needed (S3 presigned URLs).

Stripe webhooks are the canonical example:

```
Stripe-Signature: t=1719996399,v1=5257a8...,v1=alt...
```

- `t` is the timestamp.
- `v1` is `HMAC_SHA256(secret, t + "." + body)`.
- Verifier rejects requests older than 5 minutes.

---

## 8. Worked Example — Verifying a Webhook

Stripe webhook signature verification (the algorithm):

```python
import hmac, hashlib, time

def verify(payload: bytes, header: str, secret: str, tolerance: int = 300) -> bool:
    parts = dict(p.split("=", 1) for p in header.split(","))
    t = int(parts["t"])
    if abs(time.time() - t) > tolerance:
        return False   # too old / future-dated → reject

    signed = f"{t}.".encode() + payload
    expected = hmac.new(secret.encode(), signed, hashlib.sha256).hexdigest()

    return hmac.compare_digest(expected, parts["v1"])
```

Two subtleties that matter:
1. **Sign the raw body, not the parsed JSON.** Parsing reorders keys; signatures fail.
2. **Use `hmac.compare_digest`, not `==`.** Constant-time comparison defeats timing attacks.

---

## 9. AWS Signature Version 4 — A Reference Design

AWS's SigV4 is the most-deployed HMAC scheme on Earth. Worth understanding even if you don't write your own.

```
1. Canonical request:
   GET\n
   /\n
   Action=ListBuckets&Version=2010-03-31\n
   host:s3.amazonaws.com\n
   x-amz-date:20260520T120000Z\n
   \n
   host;x-amz-date\n
   <SHA256 of empty body>

2. String to sign:
   AWS4-HMAC-SHA256\n
   20260520T120000Z\n
   20260520/us-east-1/s3/aws4_request\n
   <SHA256 of canonical request>

3. Signing key (derived, not the raw secret):
   kDate    = HMAC("AWS4" + secret, "20260520")
   kRegion  = HMAC(kDate, "us-east-1")
   kService = HMAC(kRegion, "s3")
   kSigning = HMAC(kService, "aws4_request")

4. Signature: HMAC(kSigning, StringToSign)
```

Why all the layering?
- The derived signing key is bounded to **date + region + service**, so even if it leaked, it's only valid for that day, region, and service.
- The canonical request format pins every aspect of the request — query params order, headers, body hash — so nothing can be tampered with.

This complexity is the price of "key never leaves the client, requests can be sent over any channel, replay is bounded."

---

## 10. Public/Private Key Variants

For higher security, replace HMAC's shared secret with **asymmetric signing**:
- Client holds private key.
- Server holds the client's public key.
- Client signs requests with private key; server verifies with public.

Benefits:
- The server never stores anything that can forge requests.
- Compromising the server doesn't compromise clients.

Used in **JWT private_key_jwt**, **HTTP Message Signatures** (RFC 9421), **mTLS**, **Apple's APNs**, **GitHub Apps** (signed JWTs to call the API).

The downside: key management is harder. Most APIs still ship HMAC for usability.

---

## 11. API Key vs HMAC vs OAuth Token — Quick Map

| Aspect | API Key (bearer) | HMAC-signed | OAuth Access Token |
| --- | --- | --- | --- |
| Lifetime | Long | Per-request | Short |
| Replay-resistant | No (relies on TLS) | Yes (with nonce/timestamp) | No (mitigated by short TTL) |
| Tied to user identity | No | No | Yes |
| Channel security needed | TLS strongly | Less critical (still use TLS) | TLS strongly |
| Implementation complexity | Trivial | Moderate | High |
| Best for | Server-to-server, dev API consumption | Webhooks, S3-style APIs | User-delegated access |

---

## 12. Common Mistakes / Anti-Patterns

- **Storing keys in plaintext in the DB.** A leak compromises every customer's key. Hash or encrypt.
- **Logging the `Authorization` header.** Every request log line is a key. Redact at the proxy.
- **Sending API keys in URLs.** They land in browser history, Referer headers, server access logs, analytics tools. Always use headers.
- **Embedding secret keys in mobile/JS clients.** Anyone can extract them. Use a backend proxy.
- **No scoping.** A "full access" key is the AWS root key — you'll regret it. Issue restricted keys.
- **No rotation plan.** Keys outlive employees, vendors, contractors. Rotate routinely.
- **Single key per account, no overlap.** You can't rotate without downtime. Allow N keys per account.
- **Non-constant-time string comparison.** `==` on HMAC signatures leaks bits to a remote attacker. Use library `compare_digest` / `MessageDigest.isEqual`.
- **Reordering / reformatting the body before signing.** JSON serializers reorder keys, change whitespace; signatures break. Sign the raw bytes the server will receive.
- **No timestamp / nonce on signed requests.** Replay attacks possible.
- **Trusting the signature alone over HTTP (no TLS).** Signature defeats tampering and replay; TLS defeats observation. You want both.
- **Custom-rolling SigV4-style schemes.** Hard to get right. Use AWS's, or Stripe-webhook style, or HTTP Message Signatures (RFC 9421).
- **No leak-detection.** Modern services scan GitHub, paste sites, and Slack for leaked keys with known prefixes. If you don't, attackers do.

---

## 13. Cheat Card

```
API KEY     long-lived bearer string.   Authorization: Bearer sk_…
            store hashed/encrypted.   show once.   prefix for greppability.
            ALWAYS scope, restrict, rotate.

HMAC SIGN   client computes HMAC_SHA256(secret, canonical_request);
            sends signature in header.   server recomputes and compares.
            Add timestamp + nonce → replay protection.
            Compare with constant-time (compare_digest).

WHEN
  API key  → backend-to-backend with TLS, dev integrations
  HMAC    → webhooks, S3-style APIs, untrusted channels
  OAuth   → user-delegated access, federated identity

SIGNING STRING
  method + path + sorted query + selected headers + sha256(body) + timestamp
  Sign the RAW BYTES.   Never the parsed JSON.

KEY HYGIENE
  generate    secrets.token_urlsafe(32)   |   ≥128 bits
  store       bcrypt/argon2 hash OR KMS-encrypted
  display     once, then prefix-only
  rotate      90 days default; immediately after leak
  scope       narrowest permissions, ideally + IP allowlist

RULE: never put a secret key in a browser. Never log a key. Never ship raw `==`.
```

---

## 14. Resources

### Documentation
- **AWS Signature Version 4** — <https://docs.aws.amazon.com/general/latest/gr/signature-version-4.html>
- **Stripe webhook signatures** — <https://docs.stripe.com/webhooks#verify-events>
- **GitHub webhook signatures** — <https://docs.github.com/en/webhooks/using-webhooks/validating-webhook-deliveries>
- **HTTP Message Signatures (RFC 9421)** — <https://datatracker.ietf.org/doc/rfc9421/>
- **OWASP API Security Top 10** — <https://owasp.org/API-Security/>

### Articles
- "API keys vs OAuth tokens vs JSON Web Tokens" — Zapier engineering blog.
- "How Stripe Built Their API Keys" — Stripe engineering posts.
- "Restricted API keys" — Stripe docs (model implementation).
- "Why your application shouldn't trust HMAC alone" — Latacora.

### Books
- *API Security in Action* — Neil Madden. Chapters on bearer auth and request signing.

### Videos
- ByteByteGo — "API Authentication Methods".
- Hussein Nasser — "JWT vs API keys vs Sessions".

### Tools
- **GitGuardian, TruffleHog** — scan repos for leaked keys.
- **GitHub secret scanning** — built-in.
- **AWS Secrets Manager / Vault / Doppler / 1Password Secrets Automation** — key storage + rotation.

### Adjacent reading
- [Authentication vs Authorization →](./authn-vs-authz.md)
- [OAuth 2.0 & OpenID Connect →](./oauth-oidc.md)
- [JWT — JSON Web Tokens →](./jwt.md)
- [Secrets Management →](./secrets-management.md)
- [Webhooks →](../02-networking/webhooks.md)
- [Rate Limiting →](../03-apis/rate-limiting.md)

---

*Previous:* [← SSO — Single Sign-On](./sso.md)  |  *Next:* [RBAC, ABAC, ReBAC →](./access-control.md)
