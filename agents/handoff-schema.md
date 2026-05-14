# Handoff Schema

Canonical field names for inter-agent JSON handoffs in the ship pipeline. Agents' Input sections reference these names; ship skill constructs handoffs with these keys.

## Common Fields

| Field | Type | Description |
|---|---|---|
| `repo_root` | string | Absolute path to repository root |
| `artifact_dir` | string | Directory for pipeline artifacts (default: `.pipeline/`) |

## Specification Fields

| Field | Type | Description |
|---|---|---|
| `spec_file` | string | Path to spec/ticket file (planner input) |
| `content` | string | Pre-fetched spec content — alternative to `spec_file` |
| `requirements_file` | string | Path to planner output (`plan.md`) |
| `architecture_file` | string | Path to architecture design (same file as `requirements_file` for combined outputs) |

## Code Fields

| Field | Type | Description |
|---|---|---|
| `code_files` | string[] | Source files written or modified by coder |
| `test_files` | string[] | Test files written by test-writer |
| `doc_files` | string[] | Documentation files updated by doc-patcher |

## Diagnostic Fields

| Field | Type | Description |
|---|---|---|
| `failure_details` | string | Validator failure report — passed to coder when re-running after test failures |
| `review_issues` | string | Blocking reviewer issues — passed to coder when reviewer requests changes |
| `validator_notes` | string | How to run tests, env vars, fixtures — from test-writer to validator |

## Branch / PR Fields

| Field | Type | Description |
|---|---|---|
| `branch_name` | string | Git branch for the PR |
| `worktree_path` | string | Absolute path to git worktree |
| `reviewer_report` | string | Path to reviewer report file |
| `reviewer_verdict` | string | Must be `APPROVE` for pr-agent to proceed |

## Convention Fields (optional)

Pre-loaded by ship at startup, injected into handoffs; agents skip file-discovery when present.

| Field | Type | Description |
|---|---|---|
| `code_conventions` | string | Content of `.claude/skills/how-to-code/SKILL.md` |
| `test_conventions` | string | Content of `.claude/skills/how-to-test/SKILL.md` |

## Per-Agent Summary

| Agent | Required | Optional |
|---|---|---|
| `planner` | `spec_file` OR `content`, `repo_root` | `code_conventions` |
| `coder` | `requirements_file`, `architecture_file`, `repo_root` | `failure_details`, `review_issues`, `code_conventions` |
| `test-writer` | `requirements_file`, `code_files`, `repo_root` | `test_conventions` |
| `validator` | `test_files`, `repo_root`, `validator_notes` | — |
| `reviewer` | `requirements_file`, `architecture_file`, `code_files`, `test_files`, `validator_report` | `code_conventions`, `test_conventions` |
| `doc-patcher` | `code_files`, `repo_root` | — |
| `pr-agent` | `repo_root`, `branch_name`, `worktree_path`, `code_files`, `test_files`, `doc_files`, `artifact_dir`, `requirements_file`, `reviewer_report`, `reviewer_verdict` | — |
| `ticket-analyst` | `content`, `source_type` | `existing_tickets`, `repo_root` |
| `supervisor` | `pipeline_state`, `last_stage_output` | — |
