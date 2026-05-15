---
name: ship
description: Smart development orchestrator. Bootstraps the pipeline — creates worktree, writes initial .shipstate/supervisor.md, calls supervisor. Use when the user asks to "ship", "develop", "implement", "build", "fix CI", "run the pipeline", "implement this ticket", or wants to ship code. Supervisor handles all routing from there.
allowed-tools: Read, Write, Glob, Bash, Agent, WebFetch
argument-hint: [url-or-file-or-description] [branch-name]
effort: medium
---

# Ship

Bootstrap → call supervisor → relay result.

```!
echo "Repo: $(git rev-parse --show-toplevel 2>/dev/null || echo '(not a git repo)')"
echo "Branch: $(git branch --show-current)"
echo "Existing state: $(cat .shipstate/supervisor.md 2>/dev/null | head -3 || echo '(none)')"
```

## Resume

If `.shipstate/supervisor.md` exists and stage ≠ `done`:
- Print `Resuming from stage: <stage>`
- Call `supervisor` with path to `.shipstate/supervisor.md`
- Stop (skip bootstrap)

## Bootstrap (first run)

### 1 — Parse args

- `$ARGUMENTS[0]` — URL / file path / inline description
- `$ARGUMENTS[1]` — branch name (optional; default: `ben/<topic-from-spec>`)

If no arg and no existing state: stop, ask user for spec.

### 2 — Fetch spec

| Input | Action |
|-------|--------|
| `linear.app/*/issue/<ID>` or bare `ABC-123` | Fetch via `mcp__claude_ai_Linear__get_issue` → `.shipstate/spec.md`; set `linear_ticket: <ID>`, `req_check: true` |
| GitHub URL | `gh issue view <N>` or fetch page → `.shipstate/spec.md` |
| Local file path | Read → `.shipstate/spec.md` |
| Inline text | Write to `.shipstate/spec.md` |

### 3 — Worktree

Follow `@rules/worktree.md`. Derive branch from arg or spec topic (`ben/<topic>`). `repo_slug` = `owner/repo` from `git remote get-url origin`.

Create `.shipstate/` inside the worktree.

### 4 — Conventions

Check for `.claude/skills/how-to-code/SKILL.md` and `.claude/skills/how-to-test/SKILL.md`. Store paths (or `null`) for `supervisor.md`.

### 5 — Write `.shipstate/supervisor.md`

```markdown
# Pipeline State

## Stage
plan

## Spec
- file: .shipstate/spec.md
- linear_ticket: <ID or null>

## Repo
- root: <repo_root>
- worktree: <worktree_path>
- branch: <branch_name>
- slug: <owner/repo or null>

## Conventions
- code: <path or null>
- test: <path or null>

## Files
### Code

### Tests

### Docs

## Rounds
- validator: 0
- reviewer: 0

## Flags
- req_check: <true|false>
- fact_check: false
- pr_number: null
- cron_id: null
```

### 6 — Call supervisor

Call `supervisor` agent with path to `.shipstate/supervisor.md`. Relay supervisor's final output to user.

## Rules

- Never commit `.shipstate/` files
- Never push to `main`/`master`
- `.shipstate/` always lives inside the worktree, never in main checkout
