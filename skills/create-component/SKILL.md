---
name: create-component
description: Creates a new Claude Code skill or agent from scratch. Use when the user asks to "create a skill", "add a skill", "make a skill", "create an agent", "add an agent", "make an agent", or wants to automate a repeatable workflow or add a specialized pipeline role.
allowed-tools: Read Write Glob
argument-hint: [agent|skill] [name]
---

# Create Component

Create a new Claude Code skill or agent.

## Skill vs Agent

| | Skill | Agent |
|---|---|---|
| Invocation | User types `/skill-name` or context-triggered | Programmatically by orchestrator |
| Isolation | Main context by default | Runtime-isolated |
| Scope | Broad workflow | Single focused role |
| Output | Shown to user | Returned to caller |
| User-invocable | Yes | Usually no |

## Steps

1. **Detect type** — use `$ARGUMENTS[0]` if `agent` or `skill`; else ask: "Skill (user-invocable) or agent (pipeline component)?"
2. **Name and purpose** — use `$ARGUMENTS[1]` if provided; else ask: what is the single responsibility?
3. **Clarify if needed**:
   - *Skill*: trigger phrases? tools needed? effort? isolation?
   - *Agent*: inputs/outputs? tools? which pipeline calls it?
4. **Placement** — project-local (`.claude/`) if project-specific; else global (`~/.claude/`)
5. **Create files** — use [agent-template.md](agent-template.md) or [skill-template.md](skill-template.md); see [reference.md](reference.md); keep under 400 lines
6. **Confirm** — show file(s) and location; ask if adjustments needed

## Folder structure

**Skill:**
```
skills/my-skill/
├── SKILL.md        # Main instructions (required)
├── reference.md    # Reference docs — loaded when needed
├── template.md     # Template — loaded when needed
└── examples/
    └── sample.md
```

**Agent:**
```
agents/my-agent/
├── AGENT.md        # Main instructions (required)
└── reference.md    # Reference docs — loaded when needed
```
