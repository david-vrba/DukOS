# Sanity Check — Security Module

> Loaded by `sanity-check` skill on EVERY run, regardless of stack. Any FAIL in this module is automatically a BLOCKER — security vulnerabilities do not ship, ever.
>
> For each check: read the changed code and ask "can an attacker, a malicious user, or a misconfigured environment exploit this?"

---

## Security Checks

**1. Secrets and credentials in code**

The most direct path to account compromise. LLMs hardcode credentials because it produces working code immediately — they have no concept of "this will be committed."

- Hardcoded API keys, tokens, passwords, connection strings, or private keys anywhere in source files
- `.env` file itself committed to the repo (not just referenced)
- `.env*` files missing from `.gitignore` — a key removed from code but still in git history is still exposed (`git log -p --all -S "keyword"` can find it)
- Secrets in config files that get committed: `config.json`, `appsettings.json`, `application.yml`
- `grep -rn "api_key\s*=\|apiKey\s*=\|secret\s*=\|password\s*=\|token\s*=\|Bearer\s\|private_key\|sk-\|pk-\|AIza\|ghp_\|xox[bp]-" <changed files>` — flag every hit for review
- Cross-check: `cat .gitignore | grep -i "\.env\|secret\|key"` — verify sensitive files are listed

**2. Authentication gaps**

LLMs write route handlers and forget to apply auth middleware. Every new endpoint is a potential unauthenticated attack surface.

- New API routes/endpoints: trace each one — does auth middleware run before it?
- JWT implementation: missing `expiresIn` option (token never expires), secret is a weak literal like `"secret"` or `"jwt_secret"`, token signature not verified on protected routes
- Session-based auth: no `req.session?.user` or equivalent check at the top of protected handlers
- Password reset / email verification flows: token expiry enforced? Single-use tokens invalidated after use?
- `grep -n "router\.\|app\.\(get\|post\|put\|delete\|patch\)" <changed files>` — list every route, manually verify auth coverage

**3. Broken access control (IDOR — Insecure Direct Object Reference)**

LLMs write "get record by ID" without checking ownership. Any user who knows (or guesses) another user's ID can read, modify, or delete their data.

- `SELECT * FROM table WHERE id = ?` with only the ID from the request — missing `AND user_id = req.user.id` (or equivalent ownership check)
- Update/delete operations that don't verify the resource belongs to the authenticated user before modifying it
- Admin-only operations (ban user, view all records, delete any item) accessible without role check
- `grep -n "req\.params\.id\|req\.params\.\|params\[" <changed files>` — trace every param-based lookup: is ownership verified?

**4. Injection vulnerabilities**

LLMs build query strings with template literals because it looks cleaner. It's also the most exploited class of web vulnerability.

- **SQL injection**: string concatenation or template literals in queries — `WHERE id = ${req.params.id}` or `f"WHERE id = {user_id}"` — must use parameterized queries / prepared statements
- **NoSQL injection**: user input passed directly as a query filter object — `db.find(req.body)` in MongoDB allows `{ "$gt": "" }` to match everything
- **Command injection**: `child_process.exec(userInput)` / `subprocess.run(userInput, shell=True)` — never pass user-controlled strings to shell execution
- **Path traversal**: user input in file paths — `path.join(__dirname, req.params.filename)` allows `../../etc/passwd` — must sanitize and validate the resolved path stays within the allowed directory
- `grep -n "exec(\|spawn(\|query(\`\|f\"\|\.format(" <changed files>` — flag dynamic command/query construction

**5. Cross-site scripting (XSS)**

Rendering user input as HTML. LLMs use `dangerouslySetInnerHTML` freely, not understanding it bypasses React's escaping.

- `dangerouslySetInnerHTML={{ __html: userContent }}` without sanitization (use DOMPurify or equivalent first)
- `element.innerHTML = userInput` anywhere in JS — use `textContent` for plain text
- Template strings rendered into HTML: `div.innerHTML = \`<p>${user.bio}</p>\`` — user's bio could contain `<script>`
- User-controlled values used in `href` or `src` attributes without URL validation — `javascript:` protocol allows XSS via links
- `grep -n "dangerouslySetInnerHTML\|innerHTML\s*=" <changed files>` — every hit needs sanitization review

**6. CORS and security headers**

Misconfigured CORS allows any origin to make credentialed requests to your API. Missing security headers leave the browser without basic protections.

- `cors({ origin: "*" })` or `allow_origins=["*"]` in production — if credentials are involved this is a security hole; at minimum restrict to known origins
- `helmet()` missing in Express apps — adds X-Content-Type-Options, X-Frame-Options, CSP, and more
- State-changing endpoints using cookie-based auth without CSRF protection (`csurf` or SameSite cookie policy)
- Cookies set without `httpOnly` (accessible to JS), `secure` (only over HTTPS), or `sameSite` flags
- `grep -n "cors(\|allow_origins\|set_cookie\|res\.cookie(" <changed files>` — verify each

**7. Sensitive data exposure**

Data that should never leave the server leaking to clients or logs. LLMs write `SELECT *` and return the full row, including password hashes.

- Password hashes, tokens, secret fields, or internal IDs returned in API responses — always select only the fields you need, or use a serializer/DTO that explicitly excludes sensitive fields
- Full error stack traces sent to the client — in production, catch errors and return a generic message; log the full error server-side only
- Sensitive data in logs: `console.log(req.body)` / `logger.info(user)` when body/user contains passwords or tokens
- PII (email, phone, address) or session tokens in URL query parameters — these appear in server logs, browser history, and referrer headers
- `grep -n "console\.log\|logger\.\|logging\." <changed files>` — check what's being logged near auth or user data

**8. Input validation missing**

User input that goes directly into logic, storage, or system calls without validation. The boundary between "outside world" and "trusted code" must be hardened.

- Numeric inputs from requests used in calculations without type coercion + range check (e.g. negative quantity, absurdly large page size)
- String inputs stored in DB without length validation — can cause DB errors or storage bloat
- File uploads: MIME type checked by magic bytes (not just extension — extensions are user-controlled), file size limit enforced
- Email, URL, phone number fields: format validated before storage or use (invalid data causes downstream failures)
- Regex built from user input: `new RegExp(userInput)` — user can craft input that causes catastrophic backtracking (ReDoS)
- `grep -n "req\.body\|req\.params\|req\.query\|request\.json\|request\.form" <changed files>` — trace each to where it's first validated

**9. Cryptography mistakes**

Cryptography is easy to get wrong in ways that look correct. LLMs use whatever they've seen most in training data, which includes a lot of outdated patterns.

- Passwords hashed with MD5, SHA1, or plain SHA256 — these are fast hashes, not password hashes; use bcrypt, argon2, or scrypt
- `Math.random()` used to generate tokens, session IDs, reset codes, or nonces — not cryptographically secure; use `crypto.randomBytes()` (Node) or `secrets.token_hex()` (Python)
- Hardcoded encryption keys or initialization vectors (IVs) in source code — keys must come from secure config/secrets management
- IV reuse in symmetric encryption — each encryption operation needs a fresh random IV
- External API calls carrying credentials over plain HTTP (not HTTPS)
- `grep -n "Math\.random\|md5\|sha1\|hashlib\.md5\|hashlib\.sha1" <changed files>` — flag every hit

**10. Dependency vulnerabilities**

Third-party packages are the most common attack vector in modern supply chain attacks. LLMs install whatever version they know without checking for CVEs.

- Run `npm audit` / `pip-audit` / `govulncheck` against the project — note any HIGH or CRITICAL severity findings
- Lock file (`package-lock.json`, `pnpm-lock.yaml`, `yarn.lock`, `poetry.lock`) not committed → non-deterministic installs can pull vulnerable versions
- Packages installed from non-registry sources (GitHub URLs, local paths) without integrity verification
- Newly added packages: check if they're well-maintained (recent activity, not abandoned), and whether they need the permissions they request
- `grep -n "\"https://github.com\|\"file:" package.json` — flag non-registry package sources
