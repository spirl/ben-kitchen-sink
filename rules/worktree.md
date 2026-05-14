# Worktree Workflow

Before touching any file in a git repo:
1. **Find or create branch** — `git branch --list`; reuse or create `feat/<topic>` / `fix/<topic>`.
2. **Check existing worktrees** — `git worktree list`; reuse if branch already checked out.
3. **Add worktree if needed**:
   ```
   git worktree add ~/worktrees/<repo>/<branch> <branch>
   ```
   Edit under `~/worktrees/<repo>/<branch>/`, never main checkout. Use `git -C ~/worktrees/<repo>/<branch>` for all git ops.
4. **Monitor PR** — `gh pr view <number> --json state,reviewDecision,comments`; fix in same worktree, commit, push. Keep alive until merged/closed.
5. **Clean up** after merge:
   ```
   git worktree remove ~/worktrees/<repo>/<branch>
   git worktree prune
   ```

Applies to all file edits, not just dev-pipeline runs.
