---
name: log-achievement
description: Log a professional achievement to memory. Use when a significant milestone completes: PR merged, design doc finished, important ad-hoc task done. Trigger phrases: "log achievement", "record this win", "log this success".
allowed-tools: Read Write Edit
effort: low
argument-hint: brief description of what was accomplished (optional)
---

# Log Achievement Skill

Capture a professional win to the persistent achievements log.

## Steps

1. **Gather details**
   - If an argument was provided, use it as the achievement title/description.
   - Otherwise, ask the user: what was accomplished? (one sentence)
   - Ask: what was the impact or why does it matter? (one sentence, optional — skip if obvious)
   - Classify the type: PR | Design Doc | Ad-hoc | Other
   - Date defaults to today unless the user specifies otherwise.

2. **Format the entry**
   ```markdown
   ## YYYY-MM-DD — <Title>
   - **Type**: <PR | Design Doc | Ad-hoc | Other>
   - **What**: <brief description>
   - **Impact**: <what changed or improved>
   ```

3. **Append to memory file**
   - Read `/Users/ben/.claude/projects/-Users-ben/memory/user_professional_achievements.md`
   - Append the formatted entry at the end of the file
   - Write the updated file

4. **Confirm**
   - Output one line: `Logged: <Title>`
