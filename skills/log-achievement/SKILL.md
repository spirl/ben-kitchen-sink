---
name: log-achievement
description: Log a professional achievement to memory. Use when a significant milestone completes: PR merged, design doc finished, important ad-hoc task done. Trigger phrases: "log achievement", "record this win", "log this success".
allowed-tools: Read Write Edit
effort: low
argument-hint: brief description of what was accomplished (optional)
---

# Log Achievement Skill

Capture professional win to persistent achievements log.

## Steps

1. **Gather details** — use argument as title if provided; else ask what was accomplished (one sentence). Ask impact (optional, skip if obvious). Classify: PR | Design Doc | Ad-hoc | Other. Date defaults to today.

2. **Format entry**
   ```markdown
   ## YYYY-MM-DD — <Title>
   - **Type**: <PR | Design Doc | Ad-hoc | Other>
   - **What**: <brief description>
   - **Impact**: <what changed or improved>
   ```

3. **Append** — read `/Users/ben/.claude/projects/-Users-ben/memory/user_professional_achievements.md`, append entry, write file.

4. **Confirm** — output: `Logged: <Title>`
