---
name: plan-tickets
description: Transforms any input (Jira/Linear/GH issues, Google Docs, inline instructions) into clear, structured tickets. Creates or updates tickets in GitHub Issues, Linear, or Jira. Use when the user wants to clarify requirements, break down a feature, or turn vague instructions into actionable tickets.
allowed-tools: Read, Write, Glob, Bash, Agent, mcp__linear__*, mcp__jira__*, mcp__claude_ai_Google_Drive__*
argument-hint: [url-or-file-or-description]
effort: medium
---

# Plan Tickets

Turn any input into clear, structured, actionable tickets. Write back to the originating system when possible.

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

`$ARGUMENTS` — the raw input (URL, file path, or inline description).

Detect the input type:
- Starts with `https://github.com` → `gh_issue`
- Starts with `https://linear.app` → `linear`
- Contains `atlassian.net/browse` → `jira`
- Starts with `https://docs.google.com` → `google_doc`
- Is a readable file path → `file`
- Anything else → `stdin`

---

### 1. Fetch Content

Retrieve the raw content based on type:

**`gh_issue`** — use the `gh` CLI:
```
gh issue view <number> --repo <org/repo> --json title,body,comments,labels
```

**`linear`** — use Linear MCP if available; otherwise ask the user to paste the ticket content.

**`jira`** — use Jira MCP if available; otherwise ask the user to paste the ticket content.

**`google_doc`** — use Google Drive MCP if available; otherwise ask the user to paste the content.

**`file`** — read the file directly.

**`stdin`** — use `$ARGUMENTS` as-is.

If fetching fails and no fallback is possible, stop and tell the user what's needed.

---

### 2. Gather Context

- **Detect repo root** — run `git rev-parse --show-toplevel`; if inside a git repo, store as `repo_root` and pass to the analyst. If not in a repo, set `repo_root = null`.
- **Ask for related tickets** — skip if `source_type = stdin` AND input is fewer than 50 words (clearly self-contained). Otherwise ask: **"Are there existing tickets this relates to?"** — if yes, fetch those too (same method as step 1).

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

Call the `ticket-analyst` agent with `.tickets/handoff_analyst.json`. Write its output to `.tickets/proposals.md`.

**Stop if** the analyst lists clarification questions under `## Clarification Needed` — show them to the user and wait for answers before continuing.

---

### 4. Review with User

Show the ticket proposals to the user and ask:
> "Here are the proposed tickets. Should I create/update them, or would you like to adjust anything first?"

Wait for confirmation before writing anything.

---

### 5. Write Tickets

For each proposal, based on the **action** field (`create` or `update`) and the source type:

**GitHub Issues:**
```bash
# Create
gh issue create --repo <org/repo> --title "<title>" --body "<description + criteria>"

# Update
gh issue edit <number> --repo <org/repo> --title "<title>" --body "<description + criteria>"
```

**Linear** — use Linear MCP if available to create or update the issue.

**Jira** — use Jira MCP if available to create or update the issue.

**Fallback (no integration available)** — write each ticket to `.tickets/<slug>.md`. Tell the user:
- Which integrations were attempted (Linear, Jira, GitHub) and why each was unavailable
- The exact paths of the files created (e.g. `.tickets/ticket-add-dark-mode.md`)

---

### 6. Report

Tell the user what was created or updated, with links where available.
Clean up `.tickets/handoff_analyst.json` (keep `.tickets/proposals.md` as a local record).

---

## Rules

- Never create or update tickets without explicit user confirmation (step 4)
- Prefer updating an existing ticket over creating a duplicate
- If MCP integration is unavailable, fall back to local `.tickets/` files — never silently skip
- Do not push code or open PRs — this skill is planning only
- Always detect `repo_root` via `git rev-parse --show-toplevel` and pass it to the analyst
- Skip the "related tickets" question when `source_type = stdin` and input is fewer than 50 words
