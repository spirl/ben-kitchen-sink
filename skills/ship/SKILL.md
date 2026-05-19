---
name: ship
description: Smart development orchestrator. Runs only needed pipeline stages (planner → coder+test-writer → validator → reviewer → doc-patcher → pr-agent → CI), loops until CI is green or user input required. Picks up from where it left off. Use when user asks to "ship", "develop", "implement", "build", "fix CI", "run the pipeline", "implement this ticket", or wants to ship code.
allowed-tools: Read Write Glob Bash Agent CronCreate CronDelete
argument-hint: [url-or-file-or-description] [branch-name]
effort: max
---

# Ship

Assess state → pick stages → run → loop until done.

```!
echo "Repo: $(git rev-parse --show-toplevel 2>/dev/null || echo '(not a git repo)')"
echo "Branch: $(git branch --show-current 2>/dev/null || echo '(unknown)')"
echo "Pipeline state: $(cat .ship/state.json 2>/dev/null || echo '(none)')"
echo "Open PRs: $(gh pr list --json number,headRefName,statusCheckRollup 2>/dev/null | jq -c '[.[] | {number,branch:.headRefName,ci:(.statusCheckRollup // [] | map(.conclusion) | unique)}]' || echo '(none)')"
```

## Stages

| # | Stage | Agent | Skip when |
|---|-------|-------|-----------|
| 0.5 | Supervise | supervisor | not skippable |
| 1 | Plan | planner | `.ship/plan.md` exists, spec unchanged |
| 2 | Code | coder | code written for current plan |
| 2.5 | Tests | test-writer | IaC/config-only/docs; no code changes |
| 3 | Validate | validator | `.ship/validator.md` = PASS, no changes since |
| 4 | Review | reviewer | `.ship/reviewer.md` = APPROVE |
| 4.1 | Requirements check (opt.) | requirements-checker | disabled by default; enable with `enable_req_check: true` or Linear spec |
| 5 | Patch docs | doc-patcher | `.ship/coder.md` unchanged since last run |
| 5.1 | Fact-check (opt.) | fact-checker | disabled by default; enable with `enable_fact_check: true` |
| 6 | Commit + Push + Open PR | (ship) | PR already open |
| 7 | PR Lifecycle | pr-agent | PR merged or closed |

## Constants

| Constant | Value | Used in |
|---|---|---|
| `MAX_VALIDATOR_ROUNDS` | 5 | Stage 3 loop |
| `MAX_REVIEWER_RETRIES` | 1 | Stage 4 retry |

## State File — `.ship/state.json`

See `agents/ship.md` for full schema. Key fields:
`stage`, `validator_rounds`, `reviewer_retries`, `pr_number`, `pr_monitor_cron_id`, `agent_history`.

## Step 0 — Assess & Setup

**A. Arguments** (first run; ignored if state exists):
- `$ARGUMENTS[0]` — URL / file path / inline description
- `$ARGUMENTS[1]` — branch (optional)

URL → fetch GitHub/Linear/Jira → `.ship/spec.md` | file → copy | inline → write

Linear URL (`linear.app/*/issue/<ID>/...` or bare `ABC-123`): extract ticket ID, save `{ "linear_ticket_id": "<ID>", "enable_req_check": true }` to state.

**B. Detect real state** (always; trumps saved state):

| Observed | Enter stage |
|---|---|
| PR on branch, all CI green | **done** |
| PR on branch, CI failing or pending | **pr-agent** |
| `.ship/reviewer.md` = APPROVE, no PR | **docs** |
| `.ship/validator.md` = PASS, no PR | **review** |
| `.ship/coder.md` + `.ship/test-writer.md` exist | **validate** |
| `.ship/coder.md` exists, no test-writer.md | **tests** |
| `.ship/plan.md` exists | **code** |
| State = `done` | Wrap up: delete worktree, record achievement |
| Nothing | **plan** (needs spec arg) |

**C. Conventions** (once; stored in state): read `.claude/skills/how-to-code/SKILL.md` → `code_conventions`, `.claude/skills/how-to-test/SKILL.md` → `test_conventions`. Absent → `null`.

**D. Worktree** — follow `@rules/worktree.md`. Derive `<worktree_path>` as `repo_root`. Create `.ship/`.

**E. Intent** — print `Stages to run: [X, Y] | Skipping: [A (reason)]`

## Loop

All agents receive `$ARGUMENTS` = `<worktree_path>/.ship`. Agents read upstream `.ship/*.md` and write to `.ship/<name>.md`.

Before each agent, append to `agent_history`: `{ "agent": "<name>", "ran_at": "<ISO-8601 UTC>", "output": ".ship/<name>.md" }`

After each stage, call supervisor with `<worktree_path>/.ship`:
- `continue` → next stage
- `intervene(agent, instructions)` → re-run stage
- `escalate` → stop, show Diagnosis

## Stage 1 — Plan

Write spec to `.ship/spec.md`. Call `planner` with `.ship/` path.
Reads `.ship/spec.md`, writes `.ship/plan.md`. Stop if `BREAKDOWN REQUIRED` or >3 open questions.
Save: `{ "stage": "code" }`

## Stage 2 — Code

Call `coder` with `.ship/` path.
Reads `.ship/plan.md` (+ `.ship/validator.md` / `.ship/reviewer.md` if present), writes `.ship/coder.md`.
Save: `{ "stage": "tests" }`

## Stage 2.5 — Tests

Skip if IaC (Terraform/Pulumi/Helm/K8s/Ansible), config-only, or docs — save `{ "stage": "validate" }` and continue.

Call `test-writer` with `.ship/` path. Reads `.ship/plan.md` + `.ship/coder.md`, writes `.ship/test-writer.md`.
Save: `{ "stage": "validate" }`

## Stage 3 — Validator Loop (max `MAX_VALIDATOR_ROUNDS` rounds)

Call `validator` with `.ship/` path. Reads `.ship/test-writer.md`, writes `.ship/validator.md`.
- **PASS** → save `{ "stage": "review" }`, continue
- **FAIL, route=coder** → re-call `coder`, re-validate
- **FAIL, route=test-writer** → re-call `test-writer`, re-validate
- **FAIL, route=analyst** → stop, ask user
- **`MAX_VALIDATOR_ROUNDS` rounds** → stop, show `.ship/validator.md`

Save `validator_rounds` each round.

## Stage 4 — Reviewer

Call `reviewer` with `.ship/` path.
Reads `.ship/plan.md`, `.ship/coder.md`, `.ship/test-writer.md`, `.ship/validator.md`, writes `.ship/reviewer.md`.
- **APPROVE** → save `{ "stage": "docs" }`, continue
- **REQUEST CHANGES** → re-call `coder`, reset `validator_rounds`, back to stage 3. Max `MAX_REVIEWER_RETRIES` retries.

## Stage 4.1 — Requirements Check (opt.) — `enable_req_check: true` or Linear URL spec

Fetch ticket via `mcp__claude_ai_Linear__get_issue` → write to `.ship/ticket.md`.
Call `requirements-checker`. Reads `.ship/ticket.md` + `.ship/coder.md`, writes `.ship/requirements.md`.
- **PASS** → save `{ "stage": "docs" }`, continue
- **FAIL** → show gaps; ask fix or proceed. Fix: re-call `coder`, back to stage 3.

## Stage 5 — Doc Patcher

Call `doc-patcher`. Reads `.ship/coder.md`, writes `.ship/doc-patcher.md`. Save: `{ "stage": "pr" }`

## Stage 5.1 — Fact-check (opt.) — `enable_fact_check: true`

Call `fact-checker`. Save: `{ "stage": "pr" }`.

## Stage 6 — Commit + Push + Open PR

1. **Read PR content** — read `.ship/plan.md` and `.ship/reviewer.md`.
2. **Get file lists** — parse `.ship/coder.md`, `.ship/test-writer.md`, `.ship/doc-patcher.md`.
3. **Read PR template** — check `<worktree_path>/.github/pull_request_template.md`; if absent, check `<worktree_path>/.github/PULL_REQUEST_TEMPLATE/*.md`. Use verbatim, filling sections from artifacts. If none, free-form body.
4. **Commit** — `git -C <worktree_path> add <all files>` then commit with terse message: subject only (≤72 chars, imperative, no period), blank line, `Co-Authored-By:` trailer — no body paragraph.
5. **Push** — `git -C <worktree_path> push -u origin <branch_name>` (no `--force`).
6. **Open PR** — `gh pr create --draft` per `@rules/pr-style.md`; apply `type:*` label.
7. **Save**: `{ "stage": "pr-agent", "pr_number": <N>, "repo_slug": "<owner/repo>" }`

## Stage 7 — PR Lifecycle

Call `pr-agent`. Reads `.ship/state.json` for pr_number, repo_slug, branch_name, repo_root.
→ cron setup, CI + comment loop, PR metadata management.
On SUCCESS → save `{ "stage": "done" }`. On FAIL → show error, stop.

## Errors

| Situation | Action |
|---|---|
| No arg/state/context | Stop, ask for spec |
| Spec not found | Stop, tell user |
| Worktree fails | Stop, tell user |
| Planner >3 questions | Stop, show them |
| Validator `MAX_VALIDATOR_ROUNDS`× fail | Stop, show `.ship/validator.md` |
| Reviewer `MAX_REVIEWER_RETRIES+1`× changes | Stop, show `.ship/reviewer.md` |
| Agent output missing or <50 chars | Stop: "Agent <name> wrote no output to .ship/<name>.md." |
| Agent ERROR | Stop, show error |

## Self-Improver

After every user intervention, invoke `self-improver`. Disable: `enable_self_improve: false` in state.

## Rules

- Never commit `.ship/` artifacts
- Never push `main`/`master`
- PRs as `--draft`
- One-line progress after each stage
- Re-read `state.json` each loop iteration
- Stage 4.1: auto-enabled for Linear URL; or `enable_req_check: true`
- Stage 5.1: `enable_fact_check: true`
