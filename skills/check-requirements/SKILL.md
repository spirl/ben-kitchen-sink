---
name: check-requirements
description: Verifies that the current branch's code satisfies the requirements from a Linear ticket. Fetches the ticket, reads changed files, and produces a covered/missing breakdown with a PASS/FAIL verdict.
allowed-tools: Read, Write, Glob, Bash, Agent, mcp__claude_ai_Linear__get_issue
argument-hint: <ticket-id-or-url>
effort: medium
---

# Check Requirements

Fetch Linear ticket → get changed files → call requirements-checker → show verdict.

```!
echo "Branch: $(git branch --show-current 2>/dev/null || echo '(unknown)')"
echo "Repo: $(git rev-parse --show-toplevel 2>/dev/null || echo '(not a git repo)')"
```

## Step 1 — Parse Ticket ID

Extract Linear ticket ID from `$ARGUMENTS`:
- Full URL: `https://linear.app/<team>/issue/ABC-123/...` → `ABC-123`
- Short ID: `ABC-123` → `ABC-123`
- No argument: stop and ask user for ticket ID

## Step 2 — Fetch Ticket

Call `mcp__claude_ai_Linear__get_issue` with the ticket ID.

Write to `/tmp/requirements-check-ticket.md`:

```markdown
# <title>

**ID**: <identifier>
**Status**: <status>
**Description**:

<description>
```

If not found or MCP fails, stop and tell user.

## Step 3 — Get Changed Files

Run from repo root:

```bash
git diff main...HEAD --name-only 2>/dev/null || git diff HEAD~1 --name-only
```

If no changed files found, warn user and continue.

## Step 4 — Call Requirements Checker

Write handoff to `/tmp/requirements-check-handoff.json`:

```json
{
  "ticket_file": "/tmp/requirements-check-ticket.md",
  "ticket_id": "<ticket_id>",
  "repo_root": "<repo_root>",
  "code_files": ["<changed files>"],
  "branch_name": "<current_branch>"
}
```

Call agent `requirements-checker` with the handoff path.

## Step 5 — Show Results

Display full report from requirements-checker.

If **FAIL**, offer next steps:
- Fix gaps in implementation
- Run `/ship` to re-run full pipeline
- Ignore specific requirements (user decides)

If **PASS**, confirm all requirements met.
