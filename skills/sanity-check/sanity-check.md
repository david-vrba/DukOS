---
name: sanity-check
description: Post-implementation audit. Checks paths, imports, security, logic, bugs, and potential issues. Default (-l) scopes to recently changed files; -h dives the full codebase. Both check for the same categories of problems.
allowed-tools: Bash,Read,Glob,Grep,AskUserQuestion,Agent
---

# Sanity Check Protocol

## Step 0 — Determine Mode

Parse the mode from the argument passed to this skill:
- `-l` or no argument → **LOW**: scope is the recently changed files only. Run every check category scoped to those files. Loads logic, security, and performance modules but applies them to changed files only. No stack-specific module loading. Type-check only (tsc), no full test suite.
- `-h` → **HIGH**: scope is the full codebase. Load stack-specific modules. Run full test suite. For checks that require finding all usages (e.g. a changed type's call sites), grep the whole codebase.

Both modes check for the exact same categories of bugs — security, logic, silly mistakes, paths, imports, everything. The difference is only how wide you search.

State the mode: `MODE: HIGH` or `MODE: LOW`

---

## Phase 1 — Establish Ground Truth

Use the conversation context as the primary source for what was changed this session. Then verify with git if available:

```bash
git rev-parse --is-inside-work-tree 2>/dev/null && \
  git diff --name-only HEAD && \
  git status --short && \
  git log --name-only --pretty="" -5 || true
```

If the command produces no output, not a git repo — use conversation context only.

State before proceeding:
```
GOAL: <one sentence from conversation context>
CHANGED FILES: <list>
COMPLETION: <did the work cover the full goal? name any gaps>
```

Any gap between goal and changed files is a Blocker.

---

## Phase 2 — Stack Detection (HIGH only)

Skip in LOW mode.

```bash
ls go.mod pyproject.toml requirements.txt next.config.ts next.config.mjs next.config.js vite.config.ts vite.config.js 2>/dev/null
cat package.json 2>/dev/null | grep '"express"'
```

Priority: **Go > Python > Next.js > React SPA > Node/Express > Generic JS**

If no stack matches, note "unrecognized stack — universal checks only" and continue to Phase 3 without loading any stack file.

Detect your skills directory first, then load the matching stack file:

```bash
# Unix/Mac/Windows Git Bash:
echo "$HOME/.claude/skills/sanity-check-stacks"
# Windows PowerShell (if not using Git Bash):
# echo "$env:USERPROFILE\.claude\skills\sanity-check-stacks"
```

| Stack | File |
|-------|------|
| Next.js | `~/.claude/skills/sanity-check-stacks/nextjs.md` |
| Python | `~/.claude/skills/sanity-check-stacks/python.md` |
| Go | `~/.claude/skills/sanity-check-stacks/go.md` |
| React SPA | `~/.claude/skills/sanity-check-stacks/react-spa.md` |
| Node/Express | `~/.claude/skills/sanity-check-stacks/node-express.md` |

Load these conditional modules if conditions match:

| Condition | File |
|-----------|------|
| Next.js or React SPA | `~/.claude/skills/sanity-check-stacks/ui-completeness.md` |
| Changed files include migrations/, .sql, alembic/, prisma/migrations/ | `~/.claude/skills/sanity-check-stacks/database-migrations.md` |

Always load these three regardless of mode:
- `~/.claude/skills/sanity-check-stacks/logic.md`
- `~/.claude/skills/sanity-check-stacks/security.md`
- `~/.claude/skills/sanity-check-stacks/performance.md`

In LOW mode: run all their checks scoped to changed files only. In HIGH mode: run them across the full codebase.

---

## Phase 3 — Bug Checks

Run every category below in both modes. In LOW mode, scope all checks to the changed files only. In HIGH mode, expand scope where noted.

### Paths & Imports
- Every new import/require: does the target file actually exist at that path? (Glob to verify)
- Case sensitivity: `Button` vs `button` — silent on Windows, breaks on Linux
- Missing required extensions (`.tsx`, `.js`)
- Barrel file imports: does `index.ts` actually export that symbol?

### Package Manager
```bash
ls pnpm-lock.yaml yarn.lock package-lock.json bun.lockb 2>/dev/null
```
Must match all install/run commands in scripts, docs, CI config.

### Environment Variables
- New env vars in code: present in `.env.example` / `.env.template`?
- Vars used in code that exist in no `.env*` file → runtime `undefined`

### Dependencies
- Packages imported but not in `package.json` / `requirements.txt` / `go.mod`
- New packages added but lockfile not updated (install never ran)

### Leftover Artifacts
```bash
grep -n "console\.log\|debugger\|TODO\|FIXME\|xxx\|temp2\|test2" <changed files>
```

### Universal Bug Checks

Go through each item explicitly:

| # | Check | Scope |
|---|-------|-------|
| 1 | Hardcoded `localhost`/`127.0.0.1` not behind env var | changed files |
| 2 | Missing `await` on async calls | changed files |
| 3 | Wrong HTTP method at call site vs route definition | changed files |
| 4 | Type/schema changed but not all call sites updated | **HIGH: full codebase grep** / LOW: changed files only |
| 5 | Missing try/catch at I/O boundaries (fetch, DB, file, external API) | changed files |
| 6 | Copy-paste variable name left over (param name doesn't match usage) | changed files |
| 7 | Off-by-one (`arr[arr.length]`, `< len` vs `<= len`, fencepost) | changed files |
| 8 | New function defined but never called anywhere | **HIGH: full codebase grep** / LOW: changed files |
| 9 | Stub shipped as real implementation (`return null`, `pass`, `// TODO: implement`) | changed files |
| 10 | `.gitignore` gap for new secrets, build artifacts, or upload dirs | changed files |
| 11 | Merge conflict markers | `grep -r "<<<<<<\|>>>>>>\|======="` changed files |
| 12 | Race condition on shared mutable state (concurrent writes without locks) | changed files |

### Security
Check for these in all code written or modified (both modes):
- Hardcoded secrets, API keys, tokens, passwords
- Auth gaps: endpoints or operations that should require auth but don't
- Broken access control / IDOR (can user A access user B's data?)
- Injection: SQL, command, LDAP — is user input ever interpolated unsanitized?
- XSS: `dangerouslySetInnerHTML`, `innerHTML`, unescaped output
- CORS headers too permissive (`*` on credentialed routes)
- Sensitive data in logs, error messages, or client-visible responses
- Weak/missing input validation at system boundaries
- Crypto mistakes: MD5/SHA1 for passwords, rolling your own crypto, weak keys

Also run the full security module loaded in Phase 2 (both modes — LOW scoped to changed files, HIGH full codebase).

**Any security failure is automatically a Blocker.**

### Logic & Edge Cases
Check for these in all code written or modified (both modes):
- Null/undefined/empty collection not guarded before use
- Division by zero or modulo zero
- Float precision issues (`0.1 + 0.2 !== 0.3`)
- Array bounds and off-by-one (already in universal checks, double-check here)
- Infinite loops or missing recursion base cases
- State invariants violated (e.g. balance going negative, index out of sync)
- Date/time pitfalls (timezone assumptions, DST, epoch vs ms)
- Business logic edge cases specific to what was just written

Also run the full logic module loaded in Phase 2 (both modes — LOW scoped to changed files, HIGH full codebase).

### Stack-Specific Checks (HIGH only)
Run all checks from the stack file loaded in Phase 2.

### Performance
Run all checks from the performance module loaded in Phase 2 (both modes — LOW scoped to changed files, HIGH full codebase).

---

## Phase 4 — Tests

**HIGH mode** — run the test command matching the stack detected in Phase 2:

| Stack | Command |
|-------|---------|
| Go | `go test ./...` |
| Python | `pytest` |
| Next.js / React SPA / Node/Express | `npx vitest run --passWithNoTests || npx jest --passWithNoTests` then `npx tsc --noEmit` |
| Generic JS / unrecognized | `npx tsc --noEmit` |

**LOW mode** — type-check only (JS/TS projects):
```bash
npx tsc --noEmit
```
Skip silently if not a JS/TS project.

If no tests exist: run build + tsc as minimum. Note absence of tests as a finding — it's not a neutral state.

---

## Phase 5 — Report

Number issues sequentially across all tiers (1, 2, 3... not restarting per tier). Only list tiers that have issues — omit empty tiers entirely. Be specific: file path + line number on every issue.

Status: **FAIL** = any Blocker present. **WARN** = warnings or nitpicks only, no blockers. **PASS** = no issues.

```
SANITY CHECK — [HIGH/LOW] — [PASS / WARN / FAIL]
<stack> | Goal: <one sentence>

Blockers
  1. <issue> [file:line]
     → <fix>

Warnings
  2. <issue> [file:line]
     → <fix>

Nitpicks
  3. <issue> [file:line] → <fix>

Tests: <PASS (X) / FAIL (X failed) / tsc PASS / tsc FAIL / NO TESTS>
Verdict: READY / NEEDS FIXES
Auto-fixable: <numbers or none>  |  Needs your call: <numbers — one-line reason each>
```

If all clear:
```
SANITY CHECK — [HIGH/LOW] — PASS
<stack> | Goal: <one sentence>

Tests: <result>
Verdict: READY
```

---

## Rules

- **Report first, fix second.** Never auto-fix without confirmation.
- **Security failures are always Blockers.** No exceptions.
- **Every issue needs a location.** "import path wrong" is useless. "line 12 of auth.ts imports from ../../utils/hash but file is at ../utils/hash" is useful.
- **Completion gaps are Blockers.** If the goal was "add auth to 3 routes" and only 2 were touched, that's a blocker regardless of what else passes.
