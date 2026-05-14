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

Extract the Linear ticket ID from `$ARGUMENTS`:

- Full URL: `https://linear.app/<team>/issue/ABC-123/...` → `ABC-123`
- Short ID: `ABC-123` → `ABC-123`
- No argument: stop and ask user for the ticket ID

## Step 2 — Fetch Ticket from Linear

Call `mcp__claude_ai_Linear__get_issue` with the ticket ID.

Write the ticket content to `/tmp/requirements-check-ticket.md`:

```markdown
# <title>

**ID**: <identifier>
**Status**: <status>
**Description**:

<description>
```

If the ticket is not found or the MCP call fails, stop and tell the user.

## Step 3 — Get Changed Files

Run from the repo root:

```bash
git diff main...HEAD --name-only 2>/dev/null || git diff HEAD~1 --name-only
```

If no changed files found, warn the user and continue (requirements-checker will still evaluate ticket completeness).

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

Display the full report from requirements-checker.

If verdict is **FAIL**, offer next steps:
- Fix the gaps in the implementation
- Run `/ship` to re-run the full pipeline
- Ignore specific requirements (user decides)

If verdict is **PASS**, confirm all requirements are met.
