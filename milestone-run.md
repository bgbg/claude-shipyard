---
description: Execute a GitHub milestone by planning and implementing its issues in order, with parallel execution support. Use when user wants to execute a milestone, run a milestone, or implement all issues in a milestone.
---

# Milestone Run

Execute a GitHub milestone end-to-end: plan, implement, review, merge, close — fully autonomous with parallel execution.

## Phase 1: Load and Analyze

1. **Fetch milestone** — Get details and all open issues via `gh api`. If no open issues remain, skip to Phase 3 (the milestone may still need its final integration→main merge).
2. **Resolve integration branch and worktree**:
   - Parse the `Integration branch:` and `Integration worktree:` lines from the `## Execution Plan` comment posted by `/milestone-plan`.
   - If `Integration branch:` is found: verify the branch exists on `origin`. If missing locally, fetch it: `git fetch origin <branch>:<branch>`.
   - If `Integration branch:` is missing from the comment: derive expected name `milestone/<milestone-number>-<slug>` and check if it exists on `origin`. If yes, use it (and offer to update the milestone comment). If no, ask user — likely the milestone predates the integration-branch flow; offer to create it from `main` now.
   - **Resolve worktree path**: Use `Integration worktree:` line if present, else default to `.trees/milestone-<milestone-number>-<slug>`. Run `git worktree list` to check whether a worktree for the integration branch already exists.
     - If a worktree exists at any path for this branch, reuse that path (it may differ from the canonical default if the user moved it).
     - If no worktree exists, create one: `git worktree add <path> <integration-branch>`. Fetch first if needed.
   - All milestone-level git operations (merges, pushes, the final integration→main PR creation) run inside this worktree via `git -C <worktree-path>`. The project's main directory is never checked out to the integration branch.
   - Once resolved, **export `CC_BASE_BRANCH=<integration-branch>`** for all sub-command invocations in this run.
3. **Determine execution order**:
   - **Preferred**: Parse `### Wave N` structure from the same milestone comment.
   - **Fallback**: Derive from `Blocked by #N` / `Depends on #N` in issue bodies. Present derived order for user confirmation.
   - Set aside `blocked:user-action` issues.
4. **Present plan** — Show integration branch, integration worktree path, waves, blocked issues. Ask user to confirm. Once confirmed, proceed to Phase 2.

## Phase 2: Execute

5. **Process waves sequentially** — Up to 3 issues in parallel per wave via sub-agents. Track status: `pending` / `in-progress` / `done` / `blocked` / `failed`.

6. **Pre-wave sync** — Before starting each wave (including the first). All commands run inside the integration worktree via `git -C <integration-worktree-path>`; the project's main directory is never touched.
   - Fetch `main`: `git fetch origin main`.
   - Update integration worktree from remote: `git -C <integration-worktree-path> pull --ff-only origin <integration-branch>`.
   - Merge `main` into integration in the worktree: `git -C <integration-worktree-path> merge --no-edit origin/main`.
   - **On conflict**: halt the milestone, leave the conflicted state in place inside the integration worktree, print `MILESTONE HALTED: integration branch conflicts with main at <integration-worktree-path> — resolve and re-run /milestone-run`, comment on the milestone, exit.
   - **On clean merge**: `git -C <integration-worktree-path> push origin <integration-branch>`. Continue to step 7.
   - This means each wave's issue worktrees fork off the latest integration HEAD, which includes both prior waves' merges and any updates from `main`.

7. **Pre-start check** — Before starting each issue in the wave:
   - Verify `todo__<issue-number>.md` exists (should have been created by `/milestone-plan`). If missing, generate it as fallback: `/make-plan --from-issue <number> --tree --base <integration-branch>`.
   - Confirm the plan's `Base:` line matches the current integration branch. If it points to `main` or a stale branch, regenerate.
   - Re-read the issue from GitHub (`gh issue view`). Compare against the plan. If the issue was materially updated since the plan was created (new requirements, changed scope, new comments with decisions), regenerate the plan with `--base <integration-branch>`.

8. **Implement** — Run `/plan-ok <number>` (it reads the base from the plan; `CC_BASE_BRANCH` is also exported as a safety net). On external blockers: print `BLOCKED: Issue #<number> — <description>`, comment on issue, mark blocked, move on. On technical failure: retry once, then mark `failed`.

9. **Review and Merge** (issue PR → integration branch, NOT main):
   - `/git-pre-pr --base <integration-branch>` → `/git-pr --base <integration-branch> --issues "<number>"`
   - Request `/copilot-review`, then `/code-review:code-review`
   - Wait up to 10 min (check every minute) for both reviews
   - `/gh-code-review --retry` → fix issues → `/git-add-commit-push`
   - Max 2 fix-review cycles, then escalate: `NEEDS REVIEW: Issue #<number>`
   - `/git-pr-merge` → `/pr-merged` → verify issue closed
   - `/git-pr-merge` automatically checks out the PR's base (the integration branch), so no `main` checkout happens here.

10. **Post-close refresh** — After each issue is closed:
    - Re-fetch all milestone issues (`gh api`). Check whether new issues were added to the milestone.
    - If new issues found: incorporate them into the execution order, determine their wave placement based on dependencies, and run `/make-plan --from-issue <number> --tree --base <integration-branch>` for each new issue.
    - Re-validate that the remaining execution order still makes sense given completed work and any new issues.

11. **Handle blocked issues** — When unblocked, resume from where left off. When only blocked issues remain, print summary and stay alive waiting for user.

## Phase 3: Integration → Main

Run only when all milestone issues are closed.

12. **Final sync** — Repeat step 6 (merge `main` into integration inside the integration worktree). Same conflict-halt rule applies.

13. **Open integration PR** — Run from inside the integration worktree: `git -C <integration-worktree-path> ...` for any local prep, then `gh pr create --base main --head <integration-branch>` (gh works from any cwd as long as the repo is correct; pass `-R <owner>/<repo>` if needed). Title: `Milestone <number>: <title>`. Body lists all closed issues (`Closes #<n>` for each). Use a HEREDOC for the body.

14. **Final review** (optional but on by default):
    - Run `/code-review:code-review --pr <pr-number>` for a local review of the cumulative diff.
    - Optionally request `/copilot-review --pr <pr-number>`.
    - Surface findings to user. Wait for explicit user confirmation before proceeding.
    - Skip with `--no-final-review`.

15. **Merge integration → main** — `gh pr merge <pr-number> --merge` (merge commit, not squash, to preserve per-issue PR history). Requires explicit user confirmation; do not auto-merge.

16. **Cleanup** — Do NOT check out `main` in the project's main directory. The cleanup leaves the user's main directory on whatever branch they had before milestone-run started.
    - Refresh main's ref locally: `git fetch origin main` (no checkout).
    - Remove the integration worktree: `git worktree remove <integration-worktree-path>` (use `--force` only if confirmed by user).
    - Delete the integration branch locally and remotely: `git branch -d <integration-branch>` and `git push origin --delete <integration-branch>`.
    - If any other stray worktrees exist on the integration branch, remove them too.

## Phase 4: Completion

17. **Summary** — Print status (completed/blocked/failed/PRs, integration PR URL). Close milestone if all issues closed and integration merged, otherwise report what's left.

## Arguments (from {{ARGS}})
- `<number>` or `--milestone <number|title>`: Milestone identifier
- `--skip <issues>`: Comma-separated issue numbers to skip
- `--start-from <wave>`: Resume from specific wave
- `--dry-run`: Show plan without executing
- `--no-final-review`: Skip the cumulative review on the integration→main PR (Phase 3 step 14)
- `--no-merge-to-main`: Stop after closing all issues; do not open or merge the integration→main PR. Useful when the user wants to review the integration branch manually before merging.

## Constraints
- Max 3 parallel issues, max 2 fix-review cycles per issue
- Always use worktrees. Never force-push without confirmation.
- Blocked issues don't block the pipeline. Single failure doesn't stop the milestone.
- Failed dependencies mark dependents as `blocked`.
- Network/auth errors: retry once, then escalate.
- No emojis, no time estimates.
- Issue PRs target the milestone integration branch, never `main`. The integration→main merge requires explicit user confirmation and happens only after all issues are closed.
- `main`→integration auto-merge runs at every wave boundary. On conflict, halt the milestone — do not attempt automatic resolution.
- `CC_BASE_BRANCH` must be exported for the entire run so sub-commands inherit the integration branch as their base.
- The integration branch lives in a dedicated worktree (`.trees/milestone-<n>-<slug>` by default). All milestone-level git operations use `git -C <integration-worktree-path>`. The project's main directory is never checked out to the integration branch and must be left untouched by milestone-run.
