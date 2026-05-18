---
description: Request a GitHub Copilot code review on the current PR, with results posted as PR comments
---

# Copilot Review

Request a GitHub Copilot review on the current PR (or `--pr <number>`).

## Steps

1. **Detect PR** — Use `--pr <number>` or detect from current branch via `gh pr view --json number,url`. Also detect `{owner}/{repo}` from `gh repo view --json nameWithOwner`. Abort if no PR found.

2. **Check Copilot status** (run in parallel):
   - **Pending?** `gh api repos/{owner}/{repo}/pulls/{number} --jq '.requested_reviewers[] | select(.login=="Copilot") | .login'`
   - **Reviewed?** `gh api repos/{owner}/{repo}/pulls/{number}/reviews --jq '[.[] | select(.user.login=="copilot-pull-request-reviewer[bot]")] | last'`

3. **Act based on status**:
   - **Already pending**: Print "Copilot review already requested and pending." Stop.
   - **Already reviewed**: DELETE then POST requested_reviewers with `["copilot-pull-request-reviewer[bot]"]` to re-request.
   - **Neither**: POST requested_reviewers with `["copilot-pull-request-reviewer[bot]"]`.

4. **Verify** — Check if Copilot appears in requested_reviewers. If yes, done. If not, fall through to step 5.

5. **Fallback: GitHub Models API** — Write and run a Python script that:
   - Fetches PR metadata and diff via `gh pr view` and `gh pr diff`
   - Sends diff (first 15k chars) to `https://models.inference.ai.azure.com/chat/completions` using `gpt-4o` with `gh auth token` for auth
   - Posts review as PR comment prefixed with "## Copilot Review (via GitHub Models API)"

## Arguments (from {{ARGS}})
- `--pr <number>`: Specify PR number. Defaults to current branch's PR.
