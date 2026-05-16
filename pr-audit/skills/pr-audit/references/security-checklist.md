# Security checklist

This checklist defines what the security-reviewer subagent (and the inline orchestrator in security mode) looks for. The structure is **What to Flag** + **What NOT to Flag** + **Common false positives** + **Examples**.

## What to Flag

### Injection vulnerabilities

- **SQL injection**: user input concatenated into SQL strings without parameterization. Includes `WHERE`, `ORDER BY`, `LIMIT`, dynamic table or column names taken from user input.
- **Command injection**: user input passed to `exec`, `spawn`, `system`, shell strings, or any subprocess API without escaping.
- **Path traversal**: user input used in file paths (`fs.readFile`, `open()`, `os.path.join` with user data) without validation that the resolved path stays inside an allowed root.
- **Cross-site scripting (XSS)**: user input rendered into HTML without escaping; use of `dangerouslySetInnerHTML`, `v-html`, `innerHTML`, or raw template insertion of unsanitized input.
- **LDAP / XPath / NoSQL injection**: user input concatenated into query strings for those systems.
- **Prompt injection**: user input directly concatenated into an LLM prompt without instruction-level separation, especially when the LLM output then goes into `eval`, SQL, shell, or file system operations.

### Authentication and authorization

- Routes that mutate state (POST, PUT, PATCH, DELETE) without an authentication middleware reference.
- Routes under an admin path that do not verify the caller is an admin (`isAdmin`, role claim, etc.).
- JWT verification calls that pass `verify: false`, missing `algorithms` allowlist, or accept `alg: none`.
- Session tokens placed in URLs (query string or path), which leak to referers and logs.
- Authorization checks that compare user-provided role strings without normalization (`role == "admin"` where role can be `"Admin"`, `"ADMIN"`, or `" admin "`).
- Server-side data fetched scoped to the current user only when the user is identified at the route level — but the data fetch itself does not filter by user (broken-authorization-by-default).

### Secrets and credentials

- Hardcoded secrets in source: AWS access keys, GitHub tokens, OpenAI / Anthropic API keys, Stripe live secret keys, Twilio tokens, JWT signing secrets, OAuth client secrets, database passwords, SMTP credentials.
- Secrets referenced via `process.env.X` that are exposed to the client by virtue of being prefixed with `NEXT_PUBLIC_`, `VITE_`, `REACT_APP_`, `PUBLIC_`, or being placed in client-side bundles.
- `.env`, `.env.local`, `.env.production`, `.env.*` files added to the diff (these should never be committed; they should be in `.gitignore`).
- API keys in `Authorization` headers hardcoded in client-side fetch calls.

### Cryptographic issues

- Password hashing using MD5, SHA-1, or unsalted SHA-256.
- Encryption using DES, 3DES, RC4, ECB mode, or other deprecated algorithms.
- Symmetric encryption with a hardcoded or predictable IV.
- Random tokens generated with `Math.random()`, `rand()`, `time()`-based seeds, or any non-CSPRNG source.
- TLS configuration accepting protocol versions below TLS 1.2.
- HMAC verification that uses naive string comparison instead of a constant-time comparison (`==`, `===`, `strcmp`) — timing attack.

### Data exposure

- Database queries on user tables missing a `WHERE user_id = ...` clause when the route serves a specific user's data.
- API endpoints that return collections (`findMany`, `SELECT *`) without filtering by the current user, allowing one user to enumerate others.
- Personally identifiable information (PII) in error responses returned to the client.
- Sensitive data (`password_hash`, `email`, `social_security_number`, `phone_number`) in server logs.
- Public storage buckets (S3, GCS, Supabase Storage, Firebase Storage) without authorization rules visible in the diff.
- GraphQL resolvers that return user data without an authorization check.

### CSRF / CORS / cookie security

- `Access-Control-Allow-Origin: *` on routes that mutate state.
- Missing CSRF token on state-changing form endpoints not using SameSite cookies.
- Cookies with `SameSite=None` but missing `Secure` flag.
- Cookies storing session tokens without `HttpOnly`.

### Webhook handlers

- Stripe, GitHub, Twilio, Clerk, Supabase, or any third-party webhook endpoint that processes the payload without verifying the signature header.
- Webhook handlers that trust `req.body` field values to identify the user / customer without cross-checking against the verified payload.

### File operations and uploads

- File upload endpoints without size limits (allows denial of service via large files).
- File upload endpoints without MIME type or extension validation.
- User-provided filenames used directly in file system paths without sanitization.
- Image / video / audio processing that does not bound input dimensions or duration.

### Destructive database / shell operations

- `DELETE`, `UPDATE`, or `DROP` statements without a `WHERE` clause and without a clear safety wrapper (transaction with confirmation, dry-run flag, etc.).
- Migrations that drop tables or columns without a corresponding `down` migration or backup step.
- `rm -rf` patterns in shell scripts touching user-provided paths.
- Force pushes (`git push --force`) to shared branches in CI scripts.

### Dependency risk

- Newly added dependencies that are typo-squats of popular packages (`reqests` vs `requests`, `lodahs` vs `lodash`).
- Dependencies pinned to a version with a known CVE (when detectable from package name + version, e.g. `[email protected]`).
- `package.json` `postinstall` scripts that run network requests or shell commands.

## What NOT to Flag

- Theoretical risks that require unlikely preconditions ("if an attacker has root, then...").
- Defense-in-depth suggestions when primary defenses are adequate (e.g., "also add WAF rules" when input is already parameterized).
- Issues in code outside the diff. The reviewer must not speculate about code defined elsewhere.
- Missing rate limiting on internal-only endpoints or endpoints that are clearly behind authentication.
- "Consider using library X" suggestions without a concrete vulnerability tied to the current implementation.
- Style or formatting choices (single vs double quotes, indentation).
- Comments suggesting that a working pattern could be slightly more secure if rewritten — only flag if there is a concrete attack scenario.
- Test files with hardcoded test credentials (`password123` in a `.test.ts` file is fine; the same in production code is critical).

## Common false positives to actively avoid

These patterns look risky but typically are not. Do not flag them unless additional context clearly makes them dangerous.

- `crypto.randomUUID()` or `nanoid()` — these are CSPRNG-backed by default.
- Cookie used for non-session data (theme preference, locale) without `HttpOnly` — not a finding.
- Database query with `WHERE id = $1` where `id` comes from `req.params.id` — this is normal routing, not IDOR, **unless** the route also handles other users' data without auth scoping.
- `JSON.parse(req.body)` — Express body parsing happens before this, the call is redundant but not a security issue.
- Hardcoded test API keys in `examples/`, `tests/`, `cypress/`, `playwright/`, or files matching `*.test.*`.

## Severity calibration examples

- **critical**: SQL injection with user input concatenation; hardcoded `STRIPE_SECRET_KEY` in source; `DELETE FROM users` with no WHERE; missing signature verification on a Stripe webhook that updates account balance; OpenAI API key in client-side bundle.
- **warning**: missing rate limit on a public POST endpoint; CSRF cookie without `SameSite`; missing input length validation on a textarea that goes to DB; password hash using bcrypt cost 8 (too low for 2026).
- **suggestion**: opportunity to add `helmet` middleware; consider stricter Content-Security-Policy; add explicit rate limit even though framework defaults exist.
