# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository purpose

Personal Claude Code configuration: global skills, agents, rules, and settings governing Claude's behavior across all projects.

## File edits

**Worktree exception**: for simple changes in this repo, edit directly in main checkout — no worktree needed. Reserve worktrees for shared code repos.

After editing any `.claude/**/*.md` file, invoke `/bungafy` on it (`rules/bungafy-after-edit.md`).

## Architecture

```
~/.claude/
├── skills/          # User-invocable via /skill-name
├── agents/          # Orchestrated programmatically by skills
├── rules/           # Auto-applied instructions (no explicit invocation)
├── settings.json    # Global permissions, sandbox, plugins
└── settings.local.json  # Local overrides
```

### Skills vs Agents

| | Skill | Agent |
|---|---|---|
| Invocation | User types `/skill-name` or context-trigger | Programmatically by orchestrator |
| Runs in | Main conversation context | Runtime-isolated subagent |
| Output | Shown to user | Returned to caller |
| Location | `skills/<name>/SKILL.md` | `agents/<name>/AGENT.md` |

### Ship pipeline

`/ship` is the central orchestrator: **planner → coder + test-writer → validator → reviewer → doc-patcher → pr-agent**. State in `.pipeline/state.json`. `supervisor` called after every stage; `self-improver` after every user intervention.

Canonical inter-agent handoff fields: `agents/handoff-schema.md`.

### Active skills

| Skill | Purpose |
|---|---|
| `/ship` | Full dev pipeline (plan → code → test → review → PR) |
| `/create-component` | Scaffold new skill or agent |
| `/create-pr` | Branch + worktree + push + draft PR |
| `/create-rule` | Add scoped rule to `rules/` |
| `/plan-tickets` | Turn any input into Linear/GitHub/Jira tickets |
| `/check-requirements` | Verify branch against Linear ticket |
| `/bungafy` | Compress markdown to reduce tokens |
| `/log-achievement` | Append entry to achievements log |
| `/update-docs` | Sync all docs to current code state |

### Rules (always applied)

| File | Effect |
|---|---|
| `worktree.md` | All edits go through git worktree |
| `branch-name.md` | Branches use `ben/<topic>` format |
| `french.md` | Correct French errors at top of every response |
| `pr-style.md` | PR titles imperative, drafts by default, type labels required |
| `user-communications.md` | External messages in English, explicit confirmation required |
| `linear-lifecycle.md` | Move Linear issues: In Progress → In Review → Done |
| `log-achievements.md` | Offer to log significant milestones |
| `clarify-questions.md` | Use `AskUserQuestion` for clarifications |
| `no-secrets.md` | Never access tokens, keys, or credentials |
| `bungafy-after-edit.md` | Run `/bungafy` after editing `.claude/**/*.md` |

## Key settings

- **Language**: French (all responses)
- **Plugin**: `gopls-lsp` (Go LSP)
- **Sandbox**: enabled; Python denied; SSH keys read-denied
- **Allowed domains**: `github.com`, `api.github.com`, `anthropic.com`, `docs.anthropic.com`

## Adding a new component

Use `/create-component` — asks skill vs. agent, name, purpose, tool list, writes file from template. Project-specific components go under `.claude/` in target repo; global ones under `~/.claude/`.
