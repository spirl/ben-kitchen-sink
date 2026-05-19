---
name: ship
description: Smart development orchestrator. Assesses current repo state, runs only the pipeline stages actually needed (planner → coder+test-writer → validator → reviewer → doc-patcher → pr-agent → CI), and loops until CI is green or user input is required. Picks up from where it left off on subsequent invocations. Use when the user asks to "ship", "develop", "implement", "build", "fix CI", "run the pipeline", "implement this ticket", or wants to ship code.
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
| 1 | Plan | planner | `plan.md` exists, spec unchanged |
| 2 | Code + Tests | coder + test-writer | code written for current plan; test-writer skipped for IaC/config-only/docs |
| 3 | Validate | validator | last result = PASS, no changes since |
| 4 | Review | reviewer | last verdict = APPROVE |
| 4.1 | Requirements check (opt.) | requirements-checker | disabled by default; enable with `enable_req_check: true` or Linear spec |
| 5 | Patch docs | doc-patcher | no code changed since last doc run |
| 5.1 | Fact-check (opt.) | fact-checker | disabled by default; enable with `enable_fact_check: true` |
| 6 | Commit + Push + Open PR | (ship) | PR already open |
| 7 | PR Lifecycle | pr-agent | PR merged or closed |

## Constants

| Constant | Value | Used in |
|---|---|---|
| `MAX_VALIDATOR_ROUNDS` | 5 | Stage 3 loop |
| `MAX_REVIEWER_RETRIES` | 1 | Stage 4 retry |

## State File — `.ship/state.json`

Read at startup; updated after each stage.

```json
{
  "stage": "plan|code|validate|review|req-check|docs|fact-check|pr|pr-agent|done",
  "spec_file": null, "repo_root": "", "branch_name": "",
  "worktree_path": "", "pr_number": null,
  "validator_rounds": 0, "reviewer_retries": 0,
  "code_files": [], "test_files": [], "doc_files": [],
  "code_conventions": null, "test_conventions": null,
  "linear_ticket_id": null,
  "enable_req_check": false, "enable_fact_check": false,
  "repo_slug": "", "pr_monitor_cron_id": null,
  "agent_history": []
}
```

## Step 0 — Assess & Setup

**A. Arguments** (first run; ignored if state exists):
- `$ARGUMENTS[0]` — URL / file path / inline description
- `$ARGUMENTS[1]` — branch (optional)

URL → fetch GitHub/Linear/Jira → `.ship/spec.md` | file → as-is | inline → `.ship/spec.md`

Linear URL (`linear.app/*/issue/<ID>/...` or bare `ABC-123`): extract ticket ID, save `{ "linear_ticket_id": "<ID>", "enable_req_check": true }`.

**B. Detect real state** (always; trumps saved state):

| Observed | Enter stage |
|---|---|
| PR on branch, all CI green | **done** |
| PR on branch, CI failing or pending | **pr-agent** (call pr-agent with `pr_number` from state) |
| No PR, `reviewer_report.md` = APPROVE | **docs** |
| No PR, `validator_report.md` = PASS | **review** |
| Code + tests exist | **validate** |
| Only `plan.md` | **code** |
| State = `done` | Wrap up: delete worktree, record achievement |
| Nothing | **plan** (needs spec arg) |

**C. Conventions** (once; stored in state): read `.claude/skills/how-to-code/SKILL.md` → `code_conventions`, `.claude/skills/how-to-test/SKILL.md` → `test_conventions`. Absent → `null`; agents discover own.

**D. Worktree** — follow `@rules/worktree.md`. Derive `<worktree_path>` as `repo_root`. Create `.ship/`.

**E. Intent** — print `Stages to run: [X, Y] | Skipping: [A (reason)]`

## Loop

Before dispatching each agent, append to `agent_history` in state: `{ "agent": "<name>", "ran_at": "<ISO-8601 UTC>", "output": "<report-path-or-null>" }`.

After each stage, call supervisor: `{ "pipeline_state": ".ship/state.json", "last_stage_output": "<last_report_path>" }`
- `continue` → next stage
- `intervene(agent, instructions)` → call agent, re-run stage
- `escalate` → stop, show Diagnosis

Re-assess → next stage → repeat until done, blocked, or no stages remain.

## Stage 1 — Plan

```json
{ "spec_file": "<spec_file>", "content": null, "repo_root": "<repo_root>", "code_conventions": "<code_conventions>" }
```
Call `planner` → `.ship/plan.md`. Stop if ERROR, >3 questions, or `BREAKDOWN REQUIRED`.
Save: `{ "stage": "code" }`

## Stage 2 — Code + Tests

Skip `test-writer` if IaC (Terraform/Pulumi/Helm/K8s/Ansible), config-only, or docs. Testable: call `coder` + `test-writer` in parallel. Not testable: `coder` only, `test_files = []`.

```json
{ "requirements_file": ".ship/plan.md", "architecture_file": ".ship/plan.md", "repo_root": "<repo_root>", "code_conventions": "<code_conventions>" }
```
test-writer handoff (testable only):
```json
{ "requirements_file": ".ship/plan.md", "code_files": [], "repo_root": "<repo_root>", "test_conventions": "<test_conventions>" }
```
Save: `{ "stage": "validate", "code_files": [...], "test_files": [...] }`

## Stage 3 — Validator Loop (max `MAX_VALIDATOR_ROUNDS` rounds)

```json
{ "test_files": <test_files>, "repo_root": "<repo_root>", "validator_notes": "<notes>" }
```
Call `validator` → `.ship/validator_report.md`.
- **PASS** → save `{ "stage": "review" }`, continue
- **FAIL, route=coder** → add `failure_details`, re-call coder, re-validate
- **FAIL, route=test-writer** → re-call test-writer, re-validate
- **FAIL, route=analyst** → stop, ask user
- **`MAX_VALIDATOR_ROUNDS` rounds** → stop, show report

Save `validator_rounds` each round.

## Stage 4 — Reviewer

```json
{
  "requirements_file": ".ship/plan.md", "architecture_file": ".ship/plan.md",
  "code_files": <code_files>, "test_files": <test_files>,
  "validator_report": ".ship/validator_report.md",
  "code_conventions": "<code_conventions>", "test_conventions": "<test_conventions>"
}
```
Call `reviewer` → `.ship/reviewer_report.md`.
- **APPROVE** → save `{ "stage": "docs" }`, continue
- **REQUEST CHANGES** → send blocking issues to `coder`, reset validator counter, back to stage 3. Max `MAX_REVIEWER_RETRIES` retries.

## Stage 4.1 — Requirements Check (opt.) — `enable_req_check: true` or Linear URL spec

Fetch ticket via `mcp__claude_ai_Linear__get_issue` → `.ship/ticket.md`.

```json
{
  "ticket_file": ".ship/ticket.md", "ticket_id": "<linear_ticket_id>",
  "repo_root": "<repo_root>", "code_files": <code_files>, "branch_name": "<branch_name>"
}
```
Call `requirements-checker` → `.ship/requirements_report.md`.
- **PASS** → save `{ "stage": "docs" }`, continue
- **FAIL** → show gaps; ask fix or proceed. Fix: send gaps to `coder`, back to stage 3.

## Stage 5 — Doc Patcher

```json
{ "code_files": <code_files>, "repo_root": "<repo_root>" }
```
Call `doc-patcher`. Save: `{ "stage": "pr", "doc_files": [...] }`

## Stage 5.1 — Fact-check (opt.) — `enable_fact_check: true`

Call `fact-checker` with `doc_files` + `code_files`. Save: `{ "stage": "pr" }`.

## Stage 6 — Commit + Push + Open PR

1. **Read PR content** — read `requirements_file` and `reviewer_report`; extract summary and reviewer notes.
2. **Commit** — `git -C <worktree_path> add <code_files> <test_files> <doc_files>` then `git -C <worktree_path> commit -m "feat: <one-line summary>"`.
3. **Push** — `git -C <worktree_path> push -u origin <branch_name>` (no `--force`).
4. **Open PR** — `gh pr create --draft --title "<title>" --body "<body>"` per `@rules/pr-style.md`; apply `type:*` label.
5. **Save**: `{ "stage": "pr-agent", "pr_number": <N>, "repo_slug": "<owner/repo>" }`

## Stage 7 — PR Lifecycle

```json
{
  "repo_root": "<worktree_path>", "branch_name": "<branch_name>",
  "pr_number": <N>, "repo_slug": "<owner/repo>",
  "artifact_dir": ".ship"
}
```
Call `pr-agent` → cron setup, CI + comment loop, PR metadata management.

On SUCCESS → save `{ "stage": "done" }`. On FAIL → show error, stop.

## Errors

| Situation | Action |
|---|---|
| No arg/state/context | Stop, ask for spec |
| Spec not found | Stop, tell user |
| Worktree fails | Stop, tell user |
| Planner >3 questions | Stop, show them |
| Validator `MAX_VALIDATOR_ROUNDS`× fail | Stop, show report |
| Reviewer `MAX_REVIEWER_RETRIES+1`× changes | Stop, show review |
| Agent returns empty output | Stop: "Agent <name> returned no output at stage <X>. Check agent definition." |
| Agent returns malformed output | Stop: "Agent <name> returned unexpected format at stage <X>." |
| Agent ERROR | Stop, show error |

## Self-Improver

After every user intervention (permission grant, answer, re-prompt, unblock), invoke `self-improver`. Disable: `enable_self_improve: false` in state.

## Rules

- Never commit `.ship/` artifacts
- Never push `main`/`master`
- PRs as `--draft`
- One-line progress after each stage
- Re-read `state.json` each loop iteration
- Stage 4.1: auto-enabled for Linear URL; or `enable_req_check: true`
- Stage 5.1: `enable_fact_check: true`
