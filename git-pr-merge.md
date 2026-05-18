---
description: Merge the PR for current branch and clean up branches
---

# Git PR Merge

Take the last created PR for the current branch, merge it, and clean up the branch both locally and remotely.

Assume we are authenticated with GitHub. Run the commands without checking and deal with errors if they occur.

## Behavior

1) Note current branch name (for cleanup steps)
   - Run `git branch --show-current` to capture the current branch name.
   - Use this name in later cleanup steps (branch deletion, worktree removal). Validate that the user is on the correct branch.

2) Fix outstanding review comments (on by default)
   - If the conversation context contains code review comments (e.g., from `/gh-code-review`, `/code-review:code-review`, or `/copilot-review`), fix all of them before merging.
   - Commit the fixes using `/git-add-commit-push`.
   - Skip this step if `--no-fix` is passed.

3) Find and merge PR
   - Query `gh pr list --head <current-branch> --json number,state,baseRefName` for the current branch.
   - If no PR exists, abort with error message.
   - Capture `baseRefName` — this is the branch the PR merged into (may be `main`, `master`, or a milestone integration branch like `milestone/7-foo`).
   - Execute `gh pr merge <pr-number> --merge` to merge the PR.
   - If merge fails, abort with error message.

4) Post-merge cleanup (default behavior)
   - **Update the base branch locally** (the branch the PR was merged into, captured in step 3 as `<baseRefName>`). Do NOT hardcode `main`. The behavior depends on where `<baseRefName>` is currently checked out:
     - Run `git worktree list --porcelain` and look for a worktree whose `branch` is `refs/heads/<baseRefName>`.
     - **If a worktree exists for `<baseRefName>` other than the current one** (e.g. milestone integration worktree): update it in place — `git -C <other-worktree-path> pull --ff-only`. Do NOT attempt `git checkout <baseRefName>` from the current location; that would fail because the branch is checked out elsewhere.
     - **If no other worktree has `<baseRefName>` checked out**: run `git checkout <baseRefName>` followed by `git pull --ff-only` from the current location (legacy behavior for non-milestone PRs targeting `main`).
   - Remove worktrees using the merged feature branch: Run `git worktree list` and parse output to find worktrees on `<branch-name>` (the current branch from step 1, not the base), then `git worktree remove <worktree-path>` for each (non-blocking; continue if fails or no worktrees found).
   - Delete branches (run in parallel):
     - Delete local merged branch: `git branch -d <branch-name>`
     - Delete remote merged branch: `git push origin --delete <branch-name>`
   - Remove todo file if was present (non-blocking).
   - Close open PR review comment threads: Use `gh api` to resolve any open review comment conversations on the merged PR (non-blocking; continue if none found).
   - If it is clear what github issue this PR was for (e.g., from branch name or PR title), verify it was closed by the merge. If not, notify the user.

## Arguments (from {{ARGS}})
- `--no-fix`: Skip fixing review comments before merging.
- `--no-cleanup`: Skip branch deletion (both local and remote).
- `--force-delete`: Force delete local branch even if not fully merged (use `-D` instead of `-d`).

## Error handling
- If PR doesn't exist: abort with "No PR found for current branch '<branch-name>'"
- If merge fails: abort with "Failed to merge PR #<number>. Check the PR status and try again."

## Examples
- `git-pr-merge` → merge last PR for current branch and clean up
- `git-pr-merge --no-cleanup` → merge PR but keep branches
- `git-pr-merge --force-delete` → merge PR and force delete local branch

## Heuristics
- This is a fast, script-like action. Minimal validation, maximum speed.
- Use merge strategy (not squash or rebase) to preserve commit history.
- Clean up both local and remote branches by default for hygiene.
