---
description: Create a well-formed GitHub Pull Request with auto-generated title and body
---

# Git PR

Create a well-formed GitHub Pull Request for the current project, ensuring the branch is pushed, metadata is set (title, body, reviewers, labels), and handling forks/drafts intelligently.

## Behavior
1) Ensure remote and push (attempt fast path)
   - Attempt `git push -u origin <branch>` if no upstream; otherwise `git push`.
   - On failure:
     - If behind: print actionable hint to run `git pull --rebase` and retry.
     - If no upstream: suggest `git push -u origin <branch>` and retry after user action.
     - If working tree is dirty or hooks fail: show exact git error and stop without masking it.

2) If asked to detect existing PR (best-effort) (default: disabled)
   - Query `gh pr view --json number,state,url,headRefName,baseRefName,isDraft` for the branch.
   - If an open PR exists and is not draft, print URL and exit.
   - If an open PR exists and is draft, note it for later processing and continue to step 3.
   - If this lookup errors (e.g., no PR found, fork nuance), proceed to create a PR.

3) Title and body generation
   - Generate title:
     - Use last commit subject from `git log -1 --pretty=%s`
     - If branch name matches pattern `feature/123-foo` or `fix/456-bar`, extract issue number for later use
   - Generate body:
     - Concatenate all commit messages between `<base>..HEAD` using `git log --pretty=%B`
     - Never include emojis or `Signed-off-by` or any signature lines in the PR body.
     - **CRITICAL - Issue linking**: The PR body MUST include `Closes #<issue-number>` footer:
       - If `--issues` parameter provided: add `Closes #<number>` for each issue
       - Otherwise, infer issue number from branch name (e.g., `feature/123-add-auth` → `Closes #123`)
       - If no issue number can be determined, ask once whether to proceed without an issue link; abort if declined.

4) Create PR (bias for action)
   - Resolve base branch in this priority: explicit `--base <branch>` argument → `CC_BASE_BRANCH` environment variable → default `main`.
   - Execute `gh pr create` with generated `--title`, `--body`, `--base <resolved-base>`. Set `--head` for forks if needed.
   - On failure, surface the exact `gh` error; suggest likely next steps (e.g., set upstream, authenticate, fix insufficient scopes) and stop.
   - Print resulting PR URL.

5) Post-create actions — **run ALL applicable commands in parallel using separate Bash tool calls in a single message:**
   - If an existing draft PR was detected in step 2: `gh pr ready <pr-number>`
   - Remove `planning` label from linked issue(s): `gh issue edit <issue-number> --remove-label "planning"`
   - If `--comment` flag provided: `gh issue comment <issue-number> --body "<summary with PR link>"`
     - Generate the summary from commit messages without calling any tools
   - All of these are non-blocking — if any fail, log the error and continue

6) Code review (on by default, skip with `--no-code-review`)
   - Request `/copilot-review` first (remote, takes time), then immediately run `/code-review:code-review` (local — completes in chat, no waiting needed).
   - Run `/gh-code-review --sleep 15` — this polls GitHub every minute for up to 15 minutes and stops as soon as the Copilot review appears. Do NOT proceed past this point until `/gh-code-review` has completed.
   - If after 15 minutes no Copilot review has appeared on the PR, flag this to the user and ask whether to proceed with only the local review or wait longer.
   - Fix all problems found in the reviews (both local and Copilot), then commit and push via `/git-add-commit-push`.

## Heuristics
- Prefer optimistic execution; let git/gh errors drive remediation.
- If an open PR already exists, avoid duplicate creation and exit early.
- Never swallow errors; print them verbatim and stop with clear next steps.

## Examples
- `git-pr` → create PR, auto-generate title/body, request reviews.
- `git-pr --no-code-review` → create PR without requesting reviews.
- `git-pr --base "develop"` → create PR targeting the develop branch instead of main.
- `git-pr --issues "123,456"` → create PR closing issues #123 and #456.
- `git-pr --comment` → create PR and post summary comment to linked issue(s).

## Arguments (from {{ARGS}})
- `--issues "123,456"`: Explicitly specify issue numbers to close (adds `Closes #<number>` footers to PR body). If not provided, infers from branch name pattern (e.g., `feature/123-add-auth` → issue #123).
- `--base "<branch>"`: Target branch to merge into. Resolution order: this flag → `CC_BASE_BRANCH` env var → `main`.
- `--comment`: Add a summary comment to linked issue(s) with PR details. (default: skip)
- `--no-code-review`: Skip requesting code reviews after PR creation.
