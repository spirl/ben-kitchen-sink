---
name: ship
description: Smart development orchestrator. Assesses current repo state, runs only the pipeline stages actually needed (planner → coder+test-writer → validator → reviewer → doc-patcher → pr-agent → CI), and loops until CI is green or user input is required. Picks up from where it left off on subsequent invocations. Use when the user asks to "ship", "develop", "implement", "build", "fix CI", "run the pipeline", "implement this ticket", or wants to ship code.
allowed-tools: Read Write Glob Bash Agent
argument-hint: [url-or-file-or-description] [branch-name]
effort: max
---

# Ship

Assess state → pick stages → run → loop until done.

```!
echo "Repo: $(git rev-parse --show-toplevel 2>/dev/null || echo '(not a git repo)')"
echo "Branch: $(git branch --show-current 2>/dev/null || echo '(unknown)')"
echo "Pipeline state: $(cat .pipeline/state.json 2>/dev/null || echo '(none)')"
echo "Open PRs: $(gh pr list --json number,headRefName,statusCheckRollup 2>/dev/null | jq -c '[.[] | {number,branch:.headRefName,ci:(.statusCheckRollup // [] | map(.conclusion) | unique)}]' || echo '(none)')"
```

## Stages

| # | Stage | Agent | Skip when |
|---|-------|-------|-----------|
| 1 | Plan | planner | `plan.md` exists, spec unchanged |
| 2 | Code + Tests | coder + test-writer | code written for current plan; test-writer skipped for IaC/config-only/docs |
| 3 | Validate | validator | last result = PASS, no changes since |
| 4 | Review | reviewer | last verdict = APPROVE |
| 5 | Patch docs | doc-patcher | no code changed since last doc run |
| 6 | Open PR | pr-agent | PR already open on branch |
| 7 | Monitor CI | — | CI green, or no PR yet |

## Constants

| Constant | Value | Used in |
|---|---|---|
| `MAX_VALIDATOR_ROUNDS` | 5 | Stage 3 loop |
| `MAX_REVIEWER_RETRIES` | 1 | Stage 4 retry |
| `MAX_CI_FIX_ROUNDS` | 3 | Stage 7 CI fix loop |
| `CI_POLL_INTERVAL_S` | 60 | Stage 7 polling |
| `CI_POLL_MAX` | 20 | Stage 7 timeout |

## State File

`.pipeline/state.json` — read at startup, updated after each stage.

```json
{
  "stage": "plan|code|validate|review|docs|pr|ci|done",
  "spec_file": null, "repo_root": "", "branch_name": "",
  "worktree_path": "", "pr_number": null,
  "validator_rounds": 0, "reviewer_retries": 0,
  "code_files": [], "test_files": [], "doc_files": [],
  "code_conventions": null, "test_conventions": null
}
```

## Step 0 — Assess & Setup

**A. Arguments** (first run only; ignored if state exists):
- `$ARGUMENTS[0]` — URL / file path / inline description
- `$ARGUMENTS[1]` — branch name (optional)

Input: URL → fetch GitHub/Linear/Jira → `.pipeline/spec.md` | file → as-is | inline → `.pipeline/spec.md`

**B. Detect real state** (always; trumps stale saved state):

| Observed | Enter stage |
|---|---|
| PR on branch, all CI green | **done** |
| PR on branch, CI failing | **ci** (fix + re-push) |
| PR on branch, CI pending/none | **ci** (monitor) |
| No PR, `reviewer_report.md` = APPROVE | **docs** |
| No PR, `validator_report.md` = PASS | **review** |
| Code + tests exist | **validate** |
| Only `plan.md` | **code** |
| State = `done` | Monitor PR for comments until merged, then wrap up |
| Nothing | **plan** (needs spec arg) |

**Idempotency rules:**
- If `state.json` shows a stage as `done` and its output artifacts still exist → skip that stage, do not re-run
- If stage is `pr` and a PR already exists on `branch_name` → skip pr-agent, go directly to CI monitoring
- If stage is `done` → monitor the PR for new comments until it is merged, then do wrap up (delete worktree, record achievement)

**C. Pre-load conventions** (run once; stored in state):
- Read `.claude/skills/how-to-code/SKILL.md` if it exists → store content as `code_conventions`
- Read `.claude/skills/how-to-test/SKILL.md` if it exists → store content as `test_conventions`
- If absent, the field stays `null`; agents fall back to their own discovery

**D. Worktree** — follow `@rules/worktree.md`. Derive `<worktree_path>` as `repo_root`. Create `.pipeline/`.

**E. Intent** — print `Stages to run: [X, Y] | Skipping: [A (reason)]`

## Loop

After each completed stage: re-assess → next stage → repeat until CI green, blocked, or no stages remain.

## Stage 1 — Plan

```json
{ "spec_file": "<spec_file>", "content": null, "repo_root": "<repo_root>", "code_conventions": "<code_conventions>" }
```
Call `planner` → `.pipeline/plan.md`. Stop if ERROR, >3 questions, or `BREAKDOWN REQUIRED`.
Save: `{ "stage": "code" }`

## Stage 2 — Code + Tests

Assess testability from plan. Skip `test-writer` if work is primarily IaC (Terraform/Pulumi/Helm/K8s/Ansible), config-only, or docs with no unit-testable logic.

Testable: call `coder` + `test-writer` simultaneously.
Not testable: call `coder` only, set `test_files = []`.

```json
{ "requirements_file": ".pipeline/plan.md", "architecture_file": ".pipeline/plan.md", "repo_root": "<repo_root>", "code_conventions": "<code_conventions>" }
```
test-writer handoff (testable only):
```json
{ "requirements_file": ".pipeline/plan.md", "code_files": [], "repo_root": "<repo_root>", "test_conventions": "<test_conventions>" }
```
Save: `{ "stage": "validate", "code_files": [...], "test_files": [...] }`

## Stage 3 — Validator Loop (max `MAX_VALIDATOR_ROUNDS` rounds)

```json
{ "test_files": <test_files>, "repo_root": "<repo_root>", "validator_notes": "<notes>" }
```
Call `validator` → `.pipeline/validator_report.md`.
- **PASS** → save `{ "stage": "review" }`, continue
- **FAIL, route=coder** → add `failure_details`, re-call coder, re-validate
- **FAIL, route=test-writer** → re-call test-writer, re-validate
- **FAIL, route=analyst** → stop, ask user
- **`MAX_VALIDATOR_ROUNDS` rounds** → stop, show report

Save `validator_rounds` each round.

## Stage 4 — Reviewer

```json
{
  "requirements_file": ".pipeline/plan.md", "architecture_file": ".pipeline/plan.md",
  "code_files": <code_files>, "test_files": <test_files>,
  "validator_report": ".pipeline/validator_report.md",
  "code_conventions": "<code_conventions>", "test_conventions": "<test_conventions>"
}
```
Call `reviewer` → `.pipeline/reviewer_report.md`.
- **APPROVE** → save `{ "stage": "docs" }`, continue
- **REQUEST CHANGES** → send blocking issues to `coder`, reset validator counter, back to stage 3. Max `MAX_REVIEWER_RETRIES` retries.

## Stage 5 — Doc Patcher

```json
{ "code_files": <code_files>, "repo_root": "<repo_root>" }
```
Call `doc-patcher`. Save: `{ "stage": "pr", "doc_files": [...] }`

## Stage 6 — PR Agent

```json
{
  "repo_root": "<worktree_path>", "branch_name": "<branch_name>", "worktree_path": "<worktree_path>",
  "code_files": <code_files>, "test_files": <test_files>, "doc_files": <doc_files>,
  "artifact_dir": ".pipeline", "requirements_file": ".pipeline/plan.md",
  "reviewer_report": ".pipeline/reviewer_report.md", "reviewer_verdict": "APPROVE"
}
```
Call `pr-agent`. Capture `pr_number`. Save: `{ "stage": "ci", "pr_number": <number> }`

## Stage 7 — CI Monitor & Fix

Poll `gh pr view <pr_number> --json statusCheckRollup` every `CI_POLL_INTERVAL_S` s (max `CI_POLL_MAX` polls).

| CI state | Action |
|---|---|
| All green | Save `{ "stage": "done" }`, clean up worktree, report PR URL |
| Failing | `gh run view` logs → fix via `coder` → re-run stages 3–6 → re-push |
| Failing after `MAX_CI_FIX_ROUNDS` fixes | Stop, show CI log |
| Timeout | Stop, tell user to check CI |

CI fix = new validator+reviewer cycle. Preserve `validator_rounds`.

## Errors

| Situation | Action |
|---|---|
| No arg/state/context | Stop, ask for spec |
| Spec not found | Stop, tell user |
| Worktree fails | Stop, tell user |
| Planner >3 questions | Stop, show them |
| Validator `MAX_VALIDATOR_ROUNDS`× fail | Stop, show report |
| Reviewer `MAX_REVIEWER_RETRIES+1`× changes | Stop, show review |
| CI fix `MAX_CI_FIX_ROUNDS`× fail | Stop, show CI log |
| Agent returns empty output | Stop; show user: "Agent <name> returned no output at stage <X>. Check agent definition." |
| Agent returns malformed output | Stop; show user: "Agent <name> returned unexpected format at stage <X>." |
| Agent ERROR | Stop, show error |

## Rules

- Never commit `.pipeline/` artifacts
- Never push to `main`/`master`
- Create PRs as `--draft`
- One-line progress after each stage
- Re-read `state.json` at top of every loop iteration
