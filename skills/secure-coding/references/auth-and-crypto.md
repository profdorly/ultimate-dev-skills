# Authentication, authorization, and cryptography

Use the framework's or a maintained library's implementation for everything here. The details below are for choosing settings, reviewing code, and recognizing mistakes, not for writing your own.

## Password storage

Hash passwords with a slow, salted, memory-hard algorithm. Minimum settings from the [OWASP Password Storage Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html):

| Algorithm | Minimum parameters | Notes |
|---|---|---|
| Argon2id (preferred) | 19 MiB memory, 2 iterations, parallelism 1 | Raise memory and iterations as far as login latency allows |
| scrypt | N=2^17, r=8, p=1 | |
| bcrypt | work factor 10 or more | Only the first 72 bytes are used; reject or handle longer passwords explicitly |
| PBKDF2-HMAC-SHA256 | 600,000 iterations | When FIPS-140 compliance is required |

- Libraries: `argon2-cffi` or `pwdlib` (Python), `argon2` or `@node-rs/argon2` (Node), `golang.org/x/crypto/argon2` or `.../bcrypt` (Go), Spring Security `Argon2PasswordEncoder` (Java), `password_hash($p, PASSWORD_ARGON2ID)` (PHP), `has_secure_password` (Rails, bcrypt). Framework defaults (Django's hashers, ASP.NET Identity) are fine when kept current.
- Never MD5, SHA-1, or a single round of SHA-256, with or without a salt.
- Rehash on login when the stored hash uses outdated parameters.

## Comparing secrets

Compare tokens, signatures, and API keys in constant time: `hmac.compare_digest` (Python), `crypto.timingSafeEqual` (Node; the buffers must have equal length, so compare lengths first or compare fixed-length digests), `subtle.ConstantTimeCompare` (Go), `MessageDigest.isEqual` (Java), `ActiveSupport::SecurityUtils.secure_compare` or `Rack::Utils.secure_compare` (Ruby), `hash_equals` (PHP), `CryptographicOperations.FixedTimeEquals` (.NET).

## Random values

Tokens, IDs used in URLs, reset codes, and nonces need a cryptographic random source: `secrets.token_urlsafe(32)` (Python), `crypto.randomBytes(32)` or `crypto.randomUUID()` (Node), `crypto/rand` (Go; `rand.Text()` in Go 1.24+), `SecureRandom` (Java and Ruby), `random_bytes` (PHP), `RandomNumberGenerator` (.NET). Never `Math.random`, Python's `random`, Go's `math/rand`, or time-based UUIDs (v1) for anything security-relevant.

## Sessions

- Regenerate the session ID at login and at every privilege change, to prevent session fixation.
- Invalidate sessions on the server at logout and after a password change or reset.
- Cookie attributes: `HttpOnly`, `Secure`, `SameSite=Lax` (or `Strict`), a narrow `Path`, and no `Domain` unless subdomains need it. The `__Host-` name prefix enforces `Secure`, no `Domain`, and `Path=/`.
- Use both an idle timeout and an absolute timeout.
- For browser apps, server-side sessions in `HttpOnly` cookies are simpler to get right than tokens held in JavaScript.

## JWT

- Verify the signature with an explicit allowlist of algorithms. Never trust the token's `alg` header, and reject `none`.
- Verify `exp`, `nbf`, `iss`, and `aud` on every request.
- Keep access tokens short-lived (minutes), and decide how to revoke them (short lifetimes plus refresh tokens that can be revoked on the server).
- The payload is only encoded, not encrypted: anyone holding the token can read it. Keep secrets and personal data out of it.
- HMAC keys (HS256) must be at least 256 random bits. With asymmetric keys, fetch them from the issuer's JWKS and cache them; don't accept a key embedded in the token (`jwk`, `jku`, `x5u` headers) unless it matches your allowlist.

## OAuth 2 and OpenID Connect

- Use a certified client library. Use the authorization code flow with PKCE; never the implicit flow.
- Validate `state` (CSRF) and, for OIDC, `nonce`. Register exact redirect URIs, with no wildcards.
- Validate ID tokens as described for JWTs, with the client ID as the audience.
- Request the smallest scopes that work, and keep client secrets on the server.

## Password reset and account recovery

- Tokens: random, single-use, short-lived (an hour or less), and stored hashed like passwords.
- Give the same response whether or not the account exists, and keep the timing similar.
- After a reset, invalidate existing sessions and notify the user by email.
- Rate-limit reset requests per account and per IP address.

## Login protection

- Rate-limit by account and by IP address, with increasing delays rather than permanent lockouts that attackers can trigger against others.
- Generic failure messages ("invalid email or password").
- Support multi-factor authentication where the project has it; require re-authentication for sensitive actions (changing email, password, or payout details).

## Authorization

- Decide access in one place (a policy function, middleware, or the framework's permission system) and call it from every handler, including GraphQL resolvers, background jobs, file downloads, exports, and admin tools.
- Enforce object-level access in the data query itself: `WHERE id = $1 AND owner_id = $2` (or the tenant ID), not a fetch followed by a check that someone can forget.
- Deny by default; new endpoints start closed.
- Take identity and role from the server-side session or verified token, never from request parameters.
- Test it: write a test that user A gets 404 or 403 for user B's record, for each new endpoint.
- Return 404 rather than 403 when revealing that a record exists would leak information.

## CSRF

- Any state-changing request authenticated by cookies needs CSRF protection: the framework's tokens, or a custom header combined with strict CORS. `SameSite=Lax` cookies help but don't cover every case (requests from subdomains, top-level `GET`s that change state).
- Never change state on `GET`.

## CORS

- CORS relaxes the browser's same-origin policy; it isn't access control. The server still has to authenticate and authorize every request.
- Allow specific origins. Never reflect an arbitrary `Origin` back together with `Access-Control-Allow-Credentials: true`, and don't allow the `null` origin.
- `Access-Control-Allow-Origin: *` is acceptable only for public, unauthenticated resources.

## Webhooks

- Verify the provider's signature (HMAC) over the exact raw request body, before parsing JSON, with a constant-time comparison.
- Reject old timestamps to stop replays (many providers sign a timestamp; five minutes is a common tolerance).
- Process events idempotently using the event ID, and fetch authoritative state from the provider's API when the event triggers anything valuable.

## Encryption

- Use authenticated encryption through a high-level API: libsodium (`crypto_secretbox`, `crypto_box`), Python `cryptography` (`Fernet` or `AESGCM`), Web Crypto `AES-GCM`, Go `crypto/cipher` with GCM or `golang.org/x/crypto/chacha20poly1305`, Google Tink.
- Never reuse a nonce with the same key in AES-GCM; let the library generate nonces.
- Never use ECB mode, unauthenticated CBC, or MD5 or SHA-1 for security purposes. Hashing isn't encryption; HMAC provides integrity, encryption provides confidentiality, and AEAD modes provide both.
- Keys come from a key management service or secret manager, not from code or a password without a key-derivation function.
- TLS 1.2 or newer, with certificate verification on.

## Secrets

- Read secrets from environment variables or a secret manager. Commit a `.env.example` with placeholder values; keep `.env` in `.gitignore`.
- Values exposed to client code are public. Framework prefixes (`NEXT_PUBLIC_`, `VITE_`, `REACT_APP_`, `EXPO_PUBLIC_`, SvelteKit `PUBLIC_`) bundle the variable into the browser or mobile app; never use them for API secrets.
- Don't print environment variables, configuration objects, or request headers wholesale in logs or error pages.
- A secret that was committed, logged, or pasted anywhere public is compromised. It must be revoked and rotated; deleting it from history isn't enough.
- Scope API tokens to the minimum permissions and the shortest practical lifetime.
