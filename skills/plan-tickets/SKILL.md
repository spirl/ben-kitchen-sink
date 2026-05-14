---
name: plan-tickets
description: Transforms any input (Jira/Linear/GH issues, Google Docs, inline instructions) into clear, structured tickets. Creates or updates tickets in GitHub Issues, Linear, or Jira. Use when the user wants to clarify requirements, break down a feature, or turn vague instructions into actionable tickets.
allowed-tools: Read, Write, Glob, Bash, Agent, mcp__linear__*, mcp__jira__*, mcp__claude_ai_Google_Drive__*
argument-hint: [url-or-file-or-description]
effort: medium
---

# Plan Tickets

Turn any input into clear, structured, actionable tickets. Write back to originating system when possible.

## Supported Input Types

| Input | Example |
|---|---|
| GitHub issue URL | `https://github.com/org/repo/issues/42` |
| Linear ticket URL | `https://linear.app/team/issue/ENG-123` |
| Jira ticket URL | `https://company.atlassian.net/browse/ENG-123` |
| Google Doc URL | `https://docs.google.com/document/d/...` |
| Local file | `./notes/feature-request.md` |
| Ad-hoc text | `"add dark mode to the settings page"` |

---

## Steps

### 0. Parse Input

`$ARGUMENTS` — raw input (URL, file path, or inline description).

Detect type:
- Starts with `https://github.com` → `gh_issue`
- Starts with `https://linear.app` → `linear`
- Contains `atlassian.net/browse` → `jira`
- Starts with `https://docs.google.com` → `google_doc`
- Readable file path → `file`
- Anything else → `stdin`

---

### 1. Fetch Content

**`gh_issue`**:
```
gh issue view <number> --repo <org/repo> --json title,body,comments,labels
```

**`linear`** — use Linear MCP if available; else ask user to paste content.

**`jira`** — use Jira MCP if available; else ask user to paste content.

**`google_doc`** — use Google Drive MCP if available; else ask user to paste content.

**`file`** — read directly.

**`stdin`** — use `$ARGUMENTS` as-is.

If fetching fails and no fallback possible, stop and tell user what's needed.

---

### 2. Gather Context

- **Detect repo root** — `git rev-parse --show-toplevel`; store as `repo_root` or `null`.
- **Ask for related tickets** — skip if `source_type = stdin` AND input <50 words. Otherwise ask: "Are there existing tickets this relates to?" — if yes, fetch those too.

---

### 3. Analyze

Write `.tickets/handoff_analyst.json`:
```json
{
  "content": "<fetched content>",
  "source_type": "<detected type>",
  "existing_tickets": [<titles or IDs if any>],
  "repo_root": "<repo_root or null>"
}
```

Call `ticket-analyst` agent with `.tickets/handoff_analyst.json`. Write output to `.tickets/proposals.md`.

**Stop if** analyst lists questions under `## Clarification Needed` — show them, wait for answers.

---

### 4. Review with User

Show proposals and ask:
> "Here are the proposed tickets. Should I create/update them, or adjust anything first?"

Wait for confirmation before writing anything.

---

### 5. Write Tickets

For each proposal, based on `action` field (`create` or `update`) and source type:

**GitHub Issues:**
```bash
gh issue create --repo <org/repo> --title "<title>" --body "<description + criteria>"
gh issue edit <number> --repo <org/repo> --title "<title>" --body "<description + criteria>"
```

**Linear** — use Linear MCP to create or update.

**Jira** — use Jira MCP to create or update.

**Fallback** — write to `.tickets/<slug>.md`. Tell user which integrations were attempted and why unavailable.

---

### 6. Report

Tell user what was created/updated with links. Clean up `.tickets/handoff_analyst.json` (keep `.tickets/proposals.md`).

---

## Rules

- Never create/update tickets without explicit user confirmation (step 4)
- Prefer updating existing ticket over creating duplicate
- If MCP unavailable, fall back to local `.tickets/` — never silently skip
- Don't push code or open PRs — planning only
- Always detect `repo_root` via `git rev-parse --show-toplevel`
- Skip "related tickets" question when `source_type = stdin` and input <50 words
