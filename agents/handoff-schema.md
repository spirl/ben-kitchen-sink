---
name: shipstate-spec
description: Format and convention for the .shipstate/ shared pipeline directory — replaces per-stage JSON handoffs.
---

# `.shipstate/` Spec

Shared state directory for the ship pipeline. `supervisor` is the sole writer of `supervisor.md`; agents write only their own output files.

## Directory Layout

```
.shipstate/
├── supervisor.md     ← pipeline state — written only by supervisor
├── spec.md           ← original spec / ticket content
├── planner.md        ← requirements + architecture (planner)
├── coder.md          ← code summary; coder-N.md if parallel runs
├── tester.md         ← test summary + validator notes (test-writer)
├── validator.md      ← test results + routing verdict (validator)
├── reviewer.md       ← review verdict + blocking issues (reviewer)
├── requirements.md   ← requirements check result (optional)
├── docs.md           ← docs patched list (doc-patcher)
└── pr-agent.md       ← PR state, CI status, comment context (pr-agent)
```

## `supervisor.md` Format

```markdown
# Pipeline State

## Stage
<plan|code|validate|review|req-check|docs|pr|pr-agent|done>

## Spec
- file: .shipstate/spec.md
- linear_ticket: <ID or null>

## Repo
- root: <absolute path>
- worktree: <absolute path>
- branch: <branch name>
- slug: <owner/repo or null>

## Conventions
- code: <path to how-to-code SKILL.md or null>
- test: <path to how-to-test SKILL.md or null>

## Files
### Code
- path/to/file.ext

### Tests
- path/to/test.ext

### Docs
- path/to/doc.md

## Rounds
- validator: <N>
- reviewer: <N>

## Flags
- req_check: <true|false>
- fact_check: <true|false>
- pr_number: <N or null>
- cron_id: <ID or null>
```

Omit empty list sections (e.g. skip `### Tests` if no test files yet).

## Agent Convention

- `$ARGUMENTS` for every agent: path to `.shipstate/supervisor.md`
- Each agent reads `supervisor.md` for shared context (repo paths, branch, file lists, conventions)
- Each agent reads its own predecessor files from `.shipstate/`
- Each agent writes output to its designated file
- Supervisor is the **only** writer of `supervisor.md`

## Per-Agent Files

| Agent | Reads | Writes |
|-------|-------|--------|
| planner | `supervisor.md`, `spec.md` | `planner.md` |
| coder | `supervisor.md`, `planner.md`; `validator.md` on retry; `reviewer.md` on retry | `coder.md` (or `coder-N.md`) |
| test-writer | `supervisor.md`, `planner.md`, `coder.md` | `tester.md` |
| validator | `supervisor.md`, `tester.md` | `validator.md` |
| reviewer | `supervisor.md`, `planner.md` | `reviewer.md` |
| requirements-checker | `supervisor.md`, `planner.md` | `requirements.md` |
| doc-patcher | `supervisor.md` | `docs.md` |
| pr-agent | `supervisor.md` | `pr-agent.md` |
