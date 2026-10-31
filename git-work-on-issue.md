---
description: Prepare working environment for a GitHub issue
---

# Git Work on Issue

Prepare working on a GitHub issue by creating a properly named branch and annotating the issue for the current project. This command ONLY prepares the ground - it does not plan or edit code.

## Behavior
1) Fetch issue
   - Accept `--issue <number|url>` or positional `<issue>` as the first non-flag argument. If neither is provided, ask for the issue number.
   - Fetch issue and repo metadata in one call: `gh issue view <issue> --json number,title,state,labels,repository`.
   - Do not check for GitHub authentication beforehand. If authentication fails, handle the error and attempt to authenticate.
   - Extract repository info from `repository.nameWithOwner` and `repository.defaultBranchRef.name`.
   - Verify the issue is open. Abort if closed.

2) Determine base branch
   - Resolve in this priority: explicit `--base <branch>` argument → `CC_BASE_BRANCH` environment variable → repository default branch (typically `main`).
   - Fetch only the base branch: `git fetch origin <base-branch>:<base-branch>`.
   - Fast-forward the base branch to match remote.
   - **Note**: When using `--tree`, work off `origin/<resolved-base>` and ignore current branch state and uncommitted changes.

3) Ensure clean working tree
   - If creating a regular branch (not `--tree`): abort if there are uncommitted changes.
   - If using `--tree`: ignore uncommitted changes in current working tree (worktree is independent).

4) Derive branch name
   - Pattern: `[fix|develop|feature]/<issue-number>-<kebab-description>`.
   - `type` selection rules:
     - If labels contain `bug` or `bugfix` → `fix`.
     - If labels contain `feature` or `enhancement` → `feature`.
     - Else → `develop`.
   - `description` derives from the issue title or manual prompt. Sanitize to kebab-case, ASCII, max length 48.

5) Create and switch branch
   - If `--tree`:
     - Create git worktree in `.trees/<issue-number>-<type>-<kebab-description>` and work there.
     - Copy environment files to the new worktree: `.env`, `.env.local`, `.env.*` (if they exist in the main working directory).
     - Log which files were copied (or note if none exist).
     - Exclude the whole `.trees` directory from Time Machine (macOS only, idempotent, no sudo): `command -v tmutil >/dev/null && tmutil addexclusion "$(git rev-parse --show-toplevel)/.trees"`. Excluding the parent covers every worktree under it, including ones created earlier. Committed work stays backed up in the main repo's `.git`.
   - Otherwise: `git checkout <base>` then `git pull --ff-only`, then `git checkout -b <derived-branch>`.

6) Push and annotate
   - **Run all three commands in parallel using separate Bash tool calls in a single message:**
     - `git push -u origin <derived-branch>`
     - `gh issue edit <issue> --add-label "work-in-progress"`
     - `gh issue comment <issue> --body "Starting work on this in branch [<derived-branch>](https://github.com/<owner>/<repo>/tree/<derived-branch>)."`
   - Do not include any signature lines (e.g., `Signed-off-by`).

7) Locate plan
   - **Skip this step entirely if `--plan` is true (default)** — `/make-plan` in step 9 will find or create the plan, making this step redundant.
   - **Only if `--no-plan` is set**, locate existing plan:
     - **GitHub comments have precedence**: Fetch issue comments: `gh issue view <issue> --json comments`.
     - Search comments (from newest to oldest) for plan content. A plan comment typically contains markdown sections like "## Title", "## Assumptions", "## Open Questions", "## Steps", etc.
     - If a plan is found in comments, use it as the source of truth.
     - **Fallback to todo file**: If no plan found in comments, check for `todo__<issue-number>.md` file in the current directory using Read tool.
     - If plan found in either location, inform user of the plan source (GitHub comment or todo file).
     - If no plan found in either location, inform user that no plan exists yet.

8) Output
   - Print the branch name, base, and the clickable URL: `https://github.com/<owner>/<repo>/tree/<branch>`.
   - If `--tree`: Print the worktree location: `.trees/<issue-number>-<type>-<kebab-description>`.
   - If plan was found, indicate the plan source (GitHub comment or todo file).
   - Do not plan or edit code in this command. Step 9 hands off to `/make-plan` (and to `/plan-ok` with `--skip-plan`).

9) If --plan is set (Default: true) call /make-plan --from-issue <issue number> --branch <derived-branch> <either --tree or --no-tree> [--base <resolved-base> if non-default]
   - With `--skip-plan`: add `--short` to the `/make-plan` call, then run `/plan-ok <issue number>` without waiting for approval. Exception: if the short plan has an Open Question with two or more options, stop and present it.

## Arguments (from {{ARGS}})
- `--issue <number|url>`: Target issue identifier (flag form).
- `<issue>`: Positional target issue identifier (number or GitHub issue URL).
- `--tree`: Create branch in git worktree at `.trees/<issue-number>-<type>-<kebab-description>`.
- `--base <branch>`: Base branch to fork from. Resolution order: this flag → `CC_BASE_BRANCH` env var → repo default branch.
- `--skip-plan`: For trivial issues. Writes a short plan (branch strategy and acceptance criteria) and starts `/plan-ok` without waiting for plan approval.
- `--no-plan`: Do not call `/make-plan`. Only report an existing plan (step 7) and stop.

## Heuristics
- If the issue title starts with a ticket key like `ABC-123:`, strip the prefix and use the remainder for the slug, but keep the numeric GitHub issue id in the branch name.
- Collapse repeated hyphens and trim hyphens from ends when forming the slug.
- If the derived branch already exists locally or remotely, append a short timestamp suffix `-yyyymmddHHMM`.

## Examples
- `/git-work-on-issue 42` → reads issue, creates `fix/42-login-error` (if labeled `bug`), pushes, and comments with branch link.
- `/git-work-on-issue --issue 42 --tree` → same as above but creates worktree in `.trees/42-fix-login-error`.
- `/git-work-on-issue https://github.com/<owner>/<repo>/issues/42` → same as first example.
