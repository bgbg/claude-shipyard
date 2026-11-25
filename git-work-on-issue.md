---
description: Prepare working environment for a GitHub issue
---

# Git Work on Issue

Prepare working on a GitHub issue by creating a properly named branch and annotating the issue for the current project. This command prepares the ground and then hands off to `/make-plan` (step 9). It does not edit code itself.

## Behavior
1) Fetch issue
   - Accept `--issue <number|url>` or positional `<issue>` as the first non-flag argument. If neither is provided, ask for the issue number.
   - Fetch issue and repo metadata in parallel: `gh issue view <issue> --json number,title,state,labels` and `gh repo view --json nameWithOwner,defaultBranchRef`.
   - Do not check for GitHub authentication beforehand. If authentication fails, handle the error and attempt to authenticate.
   - Extract repository info from `nameWithOwner` and `defaultBranchRef.name`.
   - Verify the issue is open. Abort if closed.

2) Determine base branch
   - Resolve in this priority: explicit `--base <branch>` argument → `CC_BASE_BRANCH` environment variable → repository default branch (typically `main`).
   - Fetch only the base branch: `git fetch origin <base-branch>:<base-branch>`.
   - Fast-forward the base branch to match remote.
   - **Note**: With a worktree (the default), work off `origin/<resolved-base>` and ignore current branch state and uncommitted changes.

3) Ensure clean working tree
   - Worktree (default): ignore uncommitted changes in the current working tree (the worktree is independent).
   - With `--no-tree`: abort if there are uncommitted changes.

4) Derive branch name
   - Pattern: `[fix|develop|feature]/<issue-number>-<kebab-description>`.
   - `type` selection rules:
     - If labels contain `bug` or `bugfix` → `fix`.
     - If labels contain `feature` or `enhancement` → `feature`.
     - Else → `develop`.
   - `description` derives from the issue title or manual prompt. Sanitize to kebab-case, ASCII, max length 48.

5) Create and switch branch
   - Worktree (default):
     - Create git worktree in `.trees/<branch-slug>` and work there. `<branch-slug>` is the branch name with `/` replaced by `-` (`fix/42-login-error` → `.trees/fix-42-login-error`). `/plan-ok` reuses this worktree.
     - Copy environment files to the new worktree: `.env`, `.env.local`, `.env.*` (if they exist in the main working directory).
     - Log which files were copied (or note if none exist).
     - Exclude the whole `.trees` directory from Time Machine (macOS only, idempotent, no sudo): `command -v tmutil >/dev/null && tmutil addexclusion "$(git rev-parse --show-toplevel)/.trees"`. Excluding the parent covers every worktree under it, including ones created earlier. Committed work stays backed up in the main repo's `.git`.
   - With `--no-tree`: `git checkout <base>` then `git pull --ff-only`, then `git checkout -b <derived-branch>`.

6) Push and annotate
   - Run these three commands in parallel:
     - `git push -u origin <derived-branch>`
     - `gh issue edit <issue> --add-label "work-in-progress"`
     - `gh issue comment <issue> --body "Starting work on this in branch [<derived-branch>](https://github.com/<owner>/<repo>/tree/<derived-branch>)."`

7) Locate plan
   - **Skip this step unless `--no-plan` is set** — `/make-plan` in step 9 finds or creates the plan.
   - **Only if `--no-plan` is set**, locate existing plan:
     - **GitHub comments have precedence**: Fetch issue comments: `gh issue view <issue> --json comments`.
     - Search comments (from newest to oldest) for plan content. A plan comment typically contains markdown sections like "## Title", "## Assumptions", "## Open Questions", "## Steps", etc.
     - If a plan is found in comments, use it as the source of truth.
     - **Fallback to todo file**: If no plan found in comments, check for `todo__<issue-number>.md` file in the root of the main checkout (the first entry of `git worktree list`) using Read tool.
     - If plan found in either location, inform user of the plan source (GitHub comment or todo file).
     - If no plan found in either location, inform user that no plan exists yet.

8) Output
   - Print the branch name, base, and the clickable URL: `https://github.com/<owner>/<repo>/tree/<branch>`.
   - Unless `--no-tree`: Print the worktree location: `.trees/<branch-slug>`.
   - If plan was found, indicate the plan source (GitHub comment or todo file).
   - Do not edit code in this command. Step 9 hands off to `/make-plan` (and to `/plan-ok` with `--skip-plan` or `--auto`).

9) Unless `--no-plan` is set, call /make-plan --from-issue <issue number> --branch <derived-branch> <--no-tree only if `--no-tree` was given> [--base <resolved-base> if non-default]
   - With `--skip-plan`: add `--short` to the `/make-plan` call, then run `/plan-ok <issue number>` without waiting for approval. Exception: if the short plan has an Open Question with two or more options, stop and present it.
   - Default (no `--skip-plan`, no `--auto`): run the full `/make-plan` and stop; the user approves the plan and runs `/plan-ok`.
   - With `--auto`: run the full `/make-plan` (no `--short`). If its Open Questions section says "No questions", run `/plan-ok <issue number>` without waiting for approval; otherwise stop and present the questions.

## Arguments (from $ARGUMENTS)
- `--issue <number|url>`: Target issue identifier (flag form).
- `<issue>`: Positional target issue identifier (number or GitHub issue URL).
- `--no-tree`: Create a regular branch in the current checkout instead of a worktree. Default: worktree at `.trees/<branch-slug>`.
- `--base <branch>`: Base branch to fork from. Resolution order: this flag → `CC_BASE_BRANCH` env var → repo default branch.
- `--skip-plan`: For trivial issues. Writes a short plan (branch strategy and acceptance criteria) and starts `/plan-ok` without waiting for plan approval.
- `--auto`: Off by default. Full plan; start `/plan-ok` without waiting for approval when the plan has no open questions.
- `--no-plan`: Do not call `/make-plan`. Only report an existing plan (step 7) and stop.

## Heuristics
- If the issue title starts with a ticket key like `ABC-123:`, strip the prefix and use the remainder for the slug, but keep the numeric GitHub issue id in the branch name.
- Collapse repeated hyphens and trim hyphens from ends when forming the slug.
- If the derived branch already exists locally or remotely, append a short timestamp suffix `-yyyymmddHHMM`.

## Examples
- `/git-work-on-issue 42` → reads issue, creates `fix/42-login-error` (if labeled `bug`), pushes, comments with branch link, and creates worktree in `.trees/fix-42-login-error`.
- `/git-work-on-issue --issue 42 --no-tree` → same as above but creates the branch in the current checkout (aborts on uncommitted changes).
- `/git-work-on-issue https://github.com/<owner>/<repo>/issues/42` → same as first example.
