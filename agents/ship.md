---
name: ship
description: Pipeline reference for the ship skill — state schema, agent_history format, and common handoff fields shared across all agents.
---

# Ship Pipeline Reference

State schema and common handoff fields. Agent-specific fields in each agent's `AGENT.md`.

## Common Handoff Fields

| Field | Type | Description |
|---|---|---|
| `repo_root` | string | Absolute path to repo root |
| `artifact_dir` | string | Pipeline artifact dir (default: `.ship/`) |
| `code_conventions` | string | `.claude/skills/how-to-code/SKILL.md` — injected by ship; agents skip file discovery when present |
| `test_conventions` | string | `.claude/skills/how-to-test/SKILL.md` — same |

## State File

`.ship/state.json` — written at startup; updated after every stage.

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

### `agent_history`

Ship appends one entry before dispatching each agent:

```json
{ "agent": "<name>", "ran_at": "<ISO-8601 UTC>", "output": "<report-path-or-null>" }
```

`supervisor` reads this to detect staleness — see `supervisor/AGENT.md`.

## Pipeline Stages

| # | Stage | Agent | Skip when |
|---|-------|-------|-----------|
| 0.5 | Supervise | supervisor | not skippable |
| 1 | Plan | planner | `plan.md` exists, spec unchanged |
| 2 | Code + Tests | coder + test-writer | code written for current plan |
| 3 | Validate | validator | last result = PASS, no changes since |
| 4 | Review | reviewer | last verdict = APPROVE |
| 4.1 | Requirements check (opt.) | requirements-checker | disabled by default |
| 5 | Patch docs | doc-patcher | no code changed since last doc run |
| 5.1 | Fact-check (opt.) | fact-checker | disabled by default |
| 6 | Commit + Push + Open PR | (ship) | PR already open |
| 7 | PR Lifecycle | pr-agent | PR merged or closed |
