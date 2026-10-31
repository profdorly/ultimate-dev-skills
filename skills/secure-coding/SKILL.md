---
name: secure-coding
description: Use when writing or reviewing code that touches untrusted input, credentials, or permissions, such as endpoints, forms, uploads, webhooks, login, sessions, JWTs, authorization checks, SQL, shell commands, file paths, outbound URLs, HTML rendering, secrets, crypto, and LLM features, and for security reviews and threat models. Safe patterns per language; never disables a security control to make something work. Not for attacking systems the user doesn't own.
license: MIT
metadata:
  author: profdorly
  version: "0.1.0"
---

# Secure Coding

Most vulnerabilities come from a few habits: trusting input, building queries and commands out of strings, checking that a user is logged in but not what they may touch, and switching off a protection because it got in the way. These rules replace those habits with safe defaults.

## Rules (non-negotiable)

1. **Never disable a security control to make something work.** That includes TLS verification (`verify=False`, `rejectUnauthorized: false`, `InsecureSkipVerify: true`, `NODE_TLS_REJECT_UNAUTHORIZED=0`, `curl -k`), CSRF protection, authentication middleware, signature checks on webhooks and tokens, CORS restrictions, Content Security Policy, sandboxing, and file permissions (`chmod 777`). If a control blocks legitimate work, configure it correctly (trust the right CA, allow the specific origin) or ask.
2. **Treat all external input as untrusted:** request bodies, query strings, headers, cookies, uploaded files and their names, webhook payloads, data from other services, LLM output, and data in your own database that originally came from users. Validate it where it enters (type, length, range, format, allowlist), and encode it for the context where it's used.
3. **Never build code or queries by concatenating input.** Use parameterized queries, argument arrays for processes, and auto-escaping templates. This covers SQL, shell, HTML, file paths, URLs, LDAP, XPath, regular expressions, templates, and `eval`.
4. **Authorize every request on the server, for every object.** Check that this caller may perform this action on this specific record, not merely that someone is logged in. Deny by default. Never trust a role, user ID, price, or tenant ID sent by the client.
5. **Keep secrets out of code, logs, URLs, error messages, client bundles, and test fixtures.** Read them from the environment or a secret manager. Variables with public prefixes (`NEXT_PUBLIC_`, `VITE_`, `REACT_APP_`, `EXPO_PUBLIC_`, `PUBLIC_`) are compiled into code that ships to browsers and devices: never put a secret in one. If you find a committed secret, stop and tell the user; it must be rotated.
6. **Don't invent cryptography or authentication.** Use maintained libraries with current parameters ([references/auth-and-crypto.md](references/auth-and-crypto.md)). No homemade token formats, password hashing, or encryption schemes.
7. **Fail closed.** When validation, authentication, or a permission check errors out, deny the request.
8. **Least privilege everywhere:** database users, API token scopes, IAM roles, container users, file permissions, and the tools you give an LLM.
9. **No security bypasses for tests or development in production code paths.** A flag that skips auth "in dev" eventually ships enabled.
10. **Report what you find.** If you notice a vulnerability outside the task, tell the user privately, with the file and line and why it's exploitable. Don't silently rewrite large unrelated areas, and never test exploits against systems the user doesn't own.

## Threat-model the change in five lines

For any change that touches a trust boundary, write this into your plan before coding:

```
Asset:    refund amounts and customers' payment records
Callers:  authenticated customers (their own charges), support staff (any charge)
Input:    charge id and amount in the request body
Abuse:    refunding someone else's charge; refunding more than was charged; replaying the request
Controls: owner-or-staff check in the query; amount <= remaining; idempotency key; audit log
```

If you can't fill in "Controls" for an abuse case, that's the work still to do.

## Checks by feature

| Feature | Check |
|---|---|
| New endpoint or handler | Authentication required? Object-level authorization in the query (`WHERE id = $1 AND owner_id = $2`)? Input schema validated, with unknown fields rejected? Request body size, pagination, and rate limits? Errors that don't leak internals? |
| Login, signup, password reset | Password hashing with current parameters; generic responses that don't reveal whether an account exists; rate limiting by account and IP; session ID regenerated at login; reset tokens single-use, short-lived, and stored hashed |
| Sessions and tokens | Cookies `HttpOnly; Secure; SameSite=Lax` (or `Strict`); server-side invalidation at logout; JWTs verified with a fixed algorithm, `exp`, `iss`, and `aud`; short lifetimes |
| File upload | Size limit; type checked by content, not just extension or `Content-Type`; a generated storage name, never the user's file name; stored outside the web root or in object storage; served with `Content-Disposition: attachment` and `X-Content-Type-Options: nosniff`; archives checked for path traversal and decompressed size |
| File download or path access | Resolve the path and confirm it's inside the allowed root; check that the caller owns the file |
| Outbound requests from user-supplied URLs | Scheme allowlist; block private, loopback, link-local, and cloud metadata addresses after DNS resolution; re-check every redirect; timeouts and size limits ([references/injection-and-input.md](references/injection-and-input.md#server-side-request-forgery-ssrf)) |
| Rendering user content | Framework auto-escaping; no raw-HTML sinks with user data; sanitize with a maintained sanitizer when HTML is required; URLs limited to `http`, `https`, and `mailto` |
| Running commands | Argument arrays, no shell; `--` before user-supplied arguments; an allowlist for anything that selects the program or its options |
| Webhooks received | Verify the provider's signature over the raw body with a constant-time comparison; reject old timestamps; process idempotently |
| Admin and internal tools | Same authentication and authorization as everything else; an audit log for destructive actions; no "internal network means trusted" assumptions |
| Background jobs and queues | Re-check authorization when the job runs; don't put secrets in job payloads; make handlers idempotent |
| LLM features | Treat model output as untrusted input (never `eval` it, render it unsanitized, or execute queries or commands from it unchecked); give tools only the user's own permissions; assume anything in the prompt, including system prompts, can leak; expect prompt injection from retrieved documents, web pages, and emails |

## Output, errors, and logs

- Production error responses get a generic message and a correlation ID; details go to the server log.
- Log security events (failed logins, permission denials, password and email changes) without passwords, tokens, full card numbers, or other secrets. Mask personal data the project doesn't need in logs.
- Use structured logging, or strip CR and LF from user-controlled values, so input can't forge log lines.
- Set security headers where the framework doesn't: `Strict-Transport-Security`, `Content-Security-Policy` (at least `frame-ancestors`), and `X-Content-Type-Options: nosniff`.

## Tools

Run what the project already has, and suggest adding one when it has none: `semgrep`, `bandit` (Python), `gosec` and `govulncheck` (Go), `brakeman` (Rails), `eslint-plugin-security`, the CodeQL workflow, `gitleaks` for secrets, and the package manager's audit command. A clean scan is not proof of safety; these tools miss authorization and logic flaws entirely. For vetting and upgrading dependencies, see the dependencies skill.

## Stop and ask when

- Making something work seems to require turning off a security control.
- The task needs real credentials, production data, or access to a system you haven't been given.
- You find an exposed secret, an authorization gap, or an injection in existing code.
- A design choice has security trade-offs the user should make: token storage, session lifetime, CORS policy, what an LLM tool may do.
- You're asked to test, scan, or exploit a system and it isn't clear the user owns it or is authorized to test it.

## Final report

For changes that touch security, add:

```
Security: refund endpoint requires auth; query filters by owner_id or staff role;
          amount validated against the remaining balance; idempotency key enforced.
Checked:  no secrets added (gitleaks clean); semgrep: 0 new findings.
Open:     no rate limit on POST /refunds yet (existing gap across the API; not in scope).
```

## Tempting shortcuts

| Shortcut | Why not |
|---|---|
| "TLS verification fails; `verify=False` for now." | "For now" ships. It also lets anyone on the network read and change the traffic. Fix the certificate trust. |
| "The ID is a UUID, nobody can guess it." | IDs leak through URLs, logs, and referrers. Check ownership. |
| "It's an internal endpoint." | Internal networks get breached, and SSRF reaches internal endpoints from outside. Authenticate it. |
| "I'll sanitize the input with a regex." | Encoding depends on the output context; a regex filter misses cases. Use parameterized APIs and context-aware escaping. |
| "The frontend already checks this." | Anyone can call the API directly. Validate and authorize on the server. |
| "CORS `*` makes the error go away." | It also allows every website to call the API from users' browsers. Allow the specific origins. |
| "I'll store the token in localStorage." | Any XSS can read it. Prefer `HttpOnly` cookies with CSRF protection, or discuss the trade-off with the user. |
| "MD5 is fine, it's only for passwords." | Fast hashes are cracked at billions of guesses per second. Use Argon2id, scrypt, or bcrypt. |
| "The model will only output valid SQL." | Prompt injection makes the model output whatever the attacker wants. Constrain and validate. |

## References

| Situation | Read |
|---|---|
| SQL, shell, path traversal, SSRF, XSS, templates, deserialization, XML, open redirects, mass assignment, prototype pollution, ReDoS, CSV injection, safe code per language | [references/injection-and-input.md](references/injection-and-input.md) |
| Password hashing, sessions, JWT, OAuth, password reset, authorization patterns, CSRF, CORS, webhooks, encryption, random tokens, constant-time comparison, secrets | [references/auth-and-crypto.md](references/auth-and-crypto.md) |
