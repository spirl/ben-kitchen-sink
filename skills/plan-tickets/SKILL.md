---
name: plan-tickets
description: Transforms any input (Linear/GH issues, Google Docs, Notion RFDs, inline instructions) into clear, structured tickets. Creates or updates tickets in Linear or GitHub Issues. Use when the user wants to clarify requirements, break down a feature, or turn vague instructions into actionable tickets.
allowed-tools: Read, Write, Glob, Bash, Agent, mcp__claude_ai_Linear__*, mcp__claude_ai_Google_Drive__*, mcp__claude_ai_Notion__*
argument-hint: [url-or-file-or-description]
effort: medium
---

# Plan Tickets

Turn any input into structured, actionable tickets in Linear.

## Supported Input Types

| Input | Example |
|---|---|
| GitHub issue URL | `https://github.com/org/repo/issues/42` |
| Linear ticket URL | `https://linear.app/team/issue/ENG-123` |
| Google Doc URL | `https://docs.google.com/document/d/...` |
| Notion page URL (RFD) | `https://www.notion.so/...` |
| Local file | `./notes/feature-request.md` |
| Ad-hoc text | `"add dark mode to the settings page"` |

## Steps

### 0. Parse Input — `$ARGUMENTS` is raw input (URL, file path, or inline description).

- `https://github.com` → `gh_issue`
- `https://linear.app` → `linear`
- `https://docs.google.com` → `google_doc`
- `notion.so` or `notion.site` → `notion_rfd`
- Readable file path → `file`
- Anything else → `stdin`

### 1. Fetch Content

**`gh_issue`**:
```
gh issue view <number> --repo <org/repo> --json title,body,comments,labels
```
**`linear`** — Linear MCP if available; else ask user to paste.
**`notion_rfd`** — `mcp__claude_ai_Notion__notion-fetch` with URL.
**`file`** — read directly.
**`stdin`** — use `$ARGUMENTS` as-is.

If fetch fails with no fallback, stop and tell user what's needed.

### 2. Context

- **Repo root** — `git rev-parse --show-toplevel`; store as `repo_root` or `null`.
- **Related tickets** — skip if `source_type = stdin` and input <50 words. Otherwise ask; fetch if yes.
- **RFD coverage** — Call `rfd-checker`; save to `.tickets/rfd_coverage.md`. Show coverage verdict (PASS/FAIL + missing sections)

### 3. Analyze — write `.tickets/handoff_analyst.json`:

```json
{
  "content": "<fetched content>",
  "source_type": "<detected type>",
  "existing_tickets": [<titles or IDs if any>],
  "repo_root": "<repo_root or null>",
  "rfd_coverage": "<path to rfd_coverage.md or null>",
  "fact_check": "<path to fact_check.md or null>"
}
```

Run in parallel:
**`ticket-analyst`** — call with `.tickets/handoff_analyst.json`; write to `.tickets/proposals.md`. If `rfd_coverage` set, propose tickets only for uncovered sections.
**`fact-checker`** (only when `source_type = notion_rfd`) — write `.tickets/handoff_fact.json`:
```json
{ "primary_file": ".tickets/rfd_content.md", "reference_files": [] }
```
Call `fact-checker`; save to `.tickets/fact_check.md`.

**Stop if** analyst has `## Clarification Needed` — show questions, wait for answers. If `fact-checker` found mismatches, show alongside proposals before step 4.

### 4. Review with User — show proposals, ask to create/update or adjust. Wait for confirmation.

### 5. Write Tickets — Linear MCP (primary, always). If unavailable, write to `.tickets/<slug>.md` and tell user.

### 6. Report — tell user what was created/updated with Linear links; clean up `.tickets/handoff_analyst.json` (keep `.tickets/proposals.md`).

## Rules

- Never create/update without explicit confirmation (step 4)
- All tickets go to Linear — no Jira, no GitHub Issues
- Prefer updating existing ticket over creating duplicate
- If Linear MCP unavailable, fall back to `.tickets/` — never silently skip
- Don't push code or open PRs — planning only
- Always detect `repo_root` via `git rev-parse --show-toplevel`
- Skip "related tickets" question when `source_type = stdin` and input <50 words
