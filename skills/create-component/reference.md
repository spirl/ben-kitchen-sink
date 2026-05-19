# Component Reference

## Agent Reference

### Valid frontmatter fields

| Field | Notes |
|---|---|
| `name` | Lowercase, hyphens only. Must be unique. |
| `description` | When to delegate — front-load key trigger phrases. |
| `tools` | Allowlist. **Comma-separated**. Omit to inherit all. |
| `disallowedTools` | Denylist. Applied before `tools`. |
| `model` | `haiku`, `sonnet`, `opus`, full model ID, or `inherit`. |
| `permissionMode` | `default`, `acceptEdits`, `auto`, `dontAsk`, `bypassPermissions`, `plan`. |
| `maxTurns` | Max agentic turns before stop. |
| `effort` | `low`, `medium`, `high`, or `max`. |

**Note:** Agents use `tools:` (comma-separated), not `allowed-tools:`. No `context: fork` for agents.

### Pipeline conventions

**Input:** Orchestrator writes JSON handoff file; agent receives path as `$ARGUMENTS`.

**Output:** Agent's final message captured by caller. Always include `## Status\nSUCCESS | FAIL | ERROR`. Write large outputs to `.ship/<agent>_report.md`.

**Error handling:** Malformed input → emit `## Status\nERROR` with description and stop.

---

## Skill Reference

### Valid frontmatter fields

| Field | Notes |
|---|---|
| `name` | Lowercase, hyphens only, max 64 chars. Defaults to directory name. |
| `description` | Under 250 chars. Front-load key trigger phrases. |
| `allowed-tools` | Space-separated (e.g. `Read Write Glob`) |
| `effort` | `low`, `medium`, `high`, or `max` |
| `context` | `fork` to run in subagent |
| `argument-hint` | Shown in autocomplete (e.g. `[filename]`) |

### String substitutions

| Variable | Description |
|---|---|
| `$ARGUMENTS` | All arguments passed when invoking |
| `$ARGUMENTS[N]` | Argument by 0-based index |
| `${CLAUDE_SKILL_DIR}` | Directory containing `SKILL.md` |

### Dynamic context injection

Inline:
```markdown
Current branch: !`git branch --show-current`
```

Multi-line:
````markdown
```!
git status --short
git log --oneline -5
```
````
