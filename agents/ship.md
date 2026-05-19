---
name: ship
description: Pipeline reference for the ship skill — state schema, agent_history format, and .ship/*.md file conventions.
---

# Ship Pipeline Reference

State schema and artifact file conventions. All agents receive `$ARGUMENTS` = path to `.ship/` directory and read/write named files there.

## Agent File Conventions

| Agent | Reads from `.ship/` | Writes to `.ship/` |
|---|---|---|
| `planner` | `spec.md`, `state.json` | `plan.md` |
| `coder` | `plan.md`, `state.json`, `validator.md`?, `reviewer.md`? | `coder.md` |
| `test-writer` | `plan.md`, `coder.md`, `state.json` | `test-writer.md` |
| `validator` | `test-writer.md`, `state.json` | `validator.md` |
| `reviewer` | `plan.md`, `coder.md`, `test-writer.md`, `validator.md`, `state.json` | `reviewer.md` |
| `requirements-checker` | `ticket.md`, `coder.md`, `state.json` | `requirements.md` |
| `doc-patcher` | `coder.md`, `state.json` | `doc-patcher.md` |
| `supervisor` | `state.json`, last agent's `*.md` (from `agent_history`) | — (read-only) |
| `pr-agent` | `state.json`, `plan.md`, `reviewer.md` | updates `state.json` |

Files marked `?` are optional (read if present).

## State File — `.ship/state.json`

Written by ship at startup; updated after every stage.

```json
{
  "stage": "plan|code|validate|review|req-check|docs|fact-check|pr|pr-agent|done",
  "repo_root": "", "branch_name": "", "worktree_path": "",
  "pr_number": null, "pr_monitor_cron_id": null, "repo_slug": "",
  "validator_rounds": 0, "reviewer_retries": 0,
  "code_conventions": null, "test_conventions": null,
  "linear_ticket_id": null,
  "enable_req_check": false, "enable_fact_check": false,
  "agent_history": []
}
```

`code_conventions` / `test_conventions` — loaded once by ship at startup from `.claude/skills/how-to-code/SKILL.md` and `.claude/skills/how-to-test/SKILL.md`; stored here so agents skip file discovery.

### `agent_history`

Ship appends one entry before dispatching each agent:

```json
{ "agent": "<name>", "ran_at": "<ISO-8601 UTC>", "output": ".ship/<name>.md" }
```

`supervisor` reads this to detect staleness — see `supervisor/AGENT.md`.

## Pipeline Stages

| # | Stage | Agent | Skip when |
|---|-------|-------|-----------|
| 0.5 | Supervise | supervisor | not skippable |
| 1 | Plan | planner | `.ship/plan.md` exists, spec unchanged |
| 2 | Code | coder | code written for current plan |
| 2.5 | Tests | test-writer | IaC/config-only/docs; or no code changes |
| 3 | Validate | validator | `.ship/validator.md` = PASS, no changes since |
| 4 | Review | reviewer | `.ship/reviewer.md` = APPROVE |
| 4.1 | Requirements check (opt.) | requirements-checker | disabled by default |
| 5 | Patch docs | doc-patcher | `.ship/coder.md` unchanged since last run |
| 5.1 | Fact-check (opt.) | fact-checker | disabled by default |
| 6 | Commit + Push + Open PR | (ship) | PR already open |
| 7 | PR Lifecycle | pr-agent | PR merged or closed |
