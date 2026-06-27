# Contributing to DukOS

DukOS is a working AI agent OS built in daily production. Good contributions — new agents, better prompts, cross-platform fixes, improved docs — make the whole swarm smarter.

**Contributor tiers:**

| Tier | Who |
|---|---|
| **Hatchling** | First issue or doc fix |
| **Dabbler** | First merged PR |
| **Diver** | Active contributor (3+ PRs) |
| **The Flock** | Core maintainers |
| **Head Duck** | David (founder) |

Top contributors can have a real duck from the founder's physical collection named after them. The collection spans dozens of countries. This is not a joke.

---

## What We Want

**High value:**
- New agent prompts for useful roles (e.g., legal, healthcare info, ecommerce ops)
- Bug fixes in `run.sh`, `_shared-rules.md`, or existing prompts
- Cross-platform fixes (Linux/macOS support for scripts that assume Windows paths)
- Improved crash recovery or checkpoint reliability
- Better documentation or clearer Quick Start steps
- New built-in skills (slash commands that agents can call)

**Low priority:**
- Cosmetic README changes
- Adding runtime dependencies (DukOS has intentionally zero)
- Features that require a server or database (file-based architecture is by design)

---

## How to Contribute

### 1. Fork and clone

```bash
git clone https://github.com/david-vrba/dukos.git
cd dukos
git remote add upstream https://github.com/david-vrba/dukos.git
```

### 2. Create a branch

```bash
git checkout -b feat/new-legal-agent
# Prefixes: fix/, feat/, docs/, skill/
```

### 3. Make your change

- **Bug fix:** reproduce it, fix it, test with `bash run.sh`
- **New agent:** follow the guide below
- **New skill:** follow the skill guide below
- **Editing `_shared-rules.md`:** this file is read by every agent on every startup — open an issue first

### 4. Test it

For prompt changes, run the agent directly:

```bash
claude --print --dangerously-skip-permissions \
  "You are the [agent-name] agent. Read agents/prompts/[agent-name].md. Summarize your role in one sentence."
```

For `run.sh` changes, syntax-check it:

```bash
bash -n run.sh
```

### 5. Open a PR

- One change per PR
- Describe what changed and why
- If changing agent behavior, note which agents are affected

---

## Writing a New Agent

Every agent is a single `.md` file in `agents/prompts/`. Anatomy of a well-written agent:

```markdown
---
name: legal
role: Legal Advisor
skills: contract review, compliance, risk flags
runs: BURST / ANY
---

# Legal Agent

One paragraph: what this agent does, what it does NOT do, who it hands off to.

## Startup Sequence
Follow _shared-rules.md exactly.
Then: identify assigned project from tasks/board.md.
Load that project's context from projects/[id]/context.md if it exists.

## Scope
- What kinds of tasks this agent picks up
- What kinds it explicitly ignores

## Output Format
- Where it writes output (reports/legal/, content-queue/, etc.)
- File naming conventions
```

**Rules for new agents:**
- Must reference `_shared-rules.md` in the startup sequence — never duplicate the rules inline
- Must define a clear output path so agents don't write to random locations
- Must define what it does NOT do — scope creep between agents causes file conflicts
- Must not read project files it doesn't need (token budget discipline)

### Checklist before submitting a new agent

- [ ] Frontmatter complete (name, role, skills, runs)
- [ ] Startup sequence references `_shared-rules.md`
- [ ] Output path defined
- [ ] Scope clearly excludes overlap with existing agents
- [ ] Tested with `claude --print` locally (even a 1-turn test counts)
- [ ] Added to the agent domain map in `README.md`

---

## Writing a New Skill

Skills are slash commands that agents (and Claude Code users) can invoke. They live in `skills/` and are registered in `.claude/settings.json`.

A skill is a single `.md` file with a YAML frontmatter block:

```markdown
---
name: my-skill
description: One line — what it does and when to use it.
---

# My Skill

Instructions the agent follows when this skill is invoked.
...
```

Skills can call external tools (yt-dlp, Whisper, Playwright, APIs) — document any dependencies clearly. If a skill has a `-h` (deep) and `-l` (fast) mode, follow that pattern for consistency.

---

## Your First PR — Full Walkthrough

New to DukOS? Here's a concrete example: fixing a typo in an agent prompt.

### Step 1 — Fork and clone (see above)

### Step 2 — Branch

```bash
git checkout -b fix/typo-in-builder-prompt
```

### Step 3 — Edit

```bash
# Edit the file — use any editor
nano agents/prompts/builder.md
```

Keep it minimal. One logical change per PR.

### Step 4 — Quick test

```bash
# Verify frontmatter is intact and agent reads cleanly
claude --print --dangerously-skip-permissions \
  "You are the builder agent. Read agents/prompts/builder.md. Summarize your role in one sentence."
```

### Step 5 — Commit and push

```bash
git add agents/prompts/builder.md
git commit -m "fix: correct typo in builder prompt scope section"
git push origin fix/typo-in-builder-prompt
```

### Step 6 — Open the PR on GitHub

Fill in:
- **What** changed (one sentence)
- **Why** it matters
- **Which agents** are affected

Example:
> Fixed a typo in `agents/prompts/builder.md` — "prompts" was misspelled in the scope section. No behavior change.

### Step 7 — Respond to review

Push updates to the same branch. The PR updates automatically.

Once merged, your change runs in production on the next shift.

---

## Project Structure

```
agents/prompts/         — One .md file per agent + _shared-rules.md (read by all)
tasks/board.md          — Central task queue (orchestrator-managed)
checkpoint/             — Per-agent crash recovery state
handoff/                — Per-agent session summaries
reports/                — Agent outputs (research, SEO, finance, etc.)
content-queue/          — Content drafts
skills/                 — Built-in slash commands (/orient, /sanity-check)
tools/                  — Helper scripts (setup.sh, cost-estimate.sh, etc.)
run.sh                  — Main launch script
CLAUDE.md               — Global agent context
```

Key invariant: **agents communicate through files only.** No shared context, no message passing between processes. If two agents need to coordinate, they write to a shared file — the orchestrator reads it.

---

## Rules

- **Never edit `tasks/board.md` directly** — orchestrator-only. Propose new tasks via `tasks/requests.md`.
- **Never edit `_shared-rules.md` in a PR** — changes need an issue and maintainer review first.
- **No new root-level dependencies.** Agents use MCP servers or existing CLI tools.
- **Git hygiene:** for contributions, use clear present-tense commit messages (`fix:`, `feat:`, `docs:`).

---

## Issues

Open an issue for:
- Bugs (include OS, shell, Claude Code version)
- Feature requests (describe the use case)
- Agent prompt improvements (include before/after if proposing a rewrite)

---

## License

By contributing, you agree your work will be licensed under the [MIT License](LICENSE).

---

*Every merged contribution moves a duck character one tier up. Welcome to the pond.*
