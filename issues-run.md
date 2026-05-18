---
description: Work on a list of GitHub issues end-to-end (plan, implement, review, merge). Default sequential, `--parallel` for concurrent. Use when the user lists multiple issue numbers and wants the full cycle for each.
---

# Issues Run

Execute multiple GitHub issues end-to-end: for each issue, run the full cycle — plan, implement, code review, merge — using a dedicated branch and worktree per issue. Reuses existing skills; this command is just the orchestrator.

## Arguments (from {{ARGS}})
- `<issues>`: One or more issue numbers, space- or comma-separated (e.g., `727 782 783` or `727,782,783`). Required.
- `--parallel`: Execute issues concurrently (max 3 at a time via sub-agents). Default: sequential.
- `--no-reorder`: Keep the user-specified order. Default: allow reordering when dependencies suggest a better order.
- `--base <branch>`: Base branch for all issues. Default: `CC_BASE_BRANCH` env var, else repo default branch.
- `--no-merge`: Stop after PR is approved; do not merge. Useful when the user wants to merge manually.
- `--dry-run`: Print the planned execution order and exit without doing work.

## Behavior

### Phase 0: Validate and order
1. Parse the issue list. Abort if any value is not a positive integer.
2. Fetch each issue in parallel: `gh issue view <n> --json number,title,state,labels,body`. Abort if any issue is closed or missing.
3. **Determine order**:
   - Default order = user-specified order.
   - Unless `--no-reorder`: consider whether a different order would be better. Signals to weigh:
     - **Dependencies**: scan each issue body for `Blocked by #N` / `Depends on #N` references among the listed issues.
     - **Other considerations**: shared files/areas (do related issues together), risk (land low-risk first), size (quick wins first), or any ordering hint in the issues themselves.
     - If a better order emerges, propose it with a one-line rationale and ask for confirmation. If the user does not respond within a reasonable wait, fall back to user-specified order.
   - In `--parallel` mode: group into waves by dependency (issues with no unmet deps among the list go in wave 1, etc.). Max 3 per wave.
4. Print the resolved plan: ordered list (or waves) + base branch. If `--dry-run`, exit here.
5. **Mark all issues as in-progress upfront** (skip if `--dry-run`): in parallel, `gh issue edit <n> --add-label "work-in-progress"` for every listed issue. This makes the queue visible on GitHub before any preparation runs. Non-blocking: log any failures but continue.

### Phase 1: Plan all issues (sequential, regardless of `--parallel`)
Plans require the most user intervention (open questions, scope clarifications), so always do this serially first — get all the human-in-the-loop work out of the way before any long-running implementation starts.

6. For each issue **in order**:
   - Run `/git-work-on-issue <issue> --tree --base <base>` — creates the branch, worktree, and (because `--plan` is the default) generates the plan via `/make-plan`.
   - If `/make-plan` raises Open Questions that need answers, resolve them with the user now. Use `/say` to alert the user if they have stepped away.
   - Confirm `todo__<issue>.md` exists and its `Base:` line matches the resolved base.
7. After all plans are ready, print a one-line summary per issue (branch, worktree path, plan file).

### Phase 2: Develop → PR → Review → Merge → Cleanup
Now run the implementation cycle for each issue — sequentially by default, or in parallel waves if `--parallel` (max 3 concurrent via sub-agents). All worktrees and plans already exist from Phase 1.

For each issue:

8. **Implement**: `/plan-ok <issue>` — runs from inside the worktree. The plan's `Base:` line drives the branch base.
9. **Pre-PR check**: `/git-pre-pr`
10. **Open PR**: `/git-pr --issues "<issue>"` (with `--base <base>` if non-default)
11. **Reviews**:
    - `/copilot-review`
    - `/code-review:code-review`
    - Wait up to 15 min (poll every minute) for both to post results.
12. **Address feedback**: `/gh-code-review --retry` → fix issues → `/git-add-commit-push`.
    - Max 2 fix-review cycles. After that, escalate via `/say`.
13. **Merge and cleanup** (skip if `--no-merge`): `/git-pr-merge` → `/pr-merged`. Verify the issue is closed and the worktree is removed.

### Phase 3: Human intervention (applies throughout)
- Whenever the run hits a blocker that needs the user (unanswered plan questions, failing tests after retries, unresolved review feedback, merge conflicts, auth failures), call `/say` with a short message naming the issue and the blocker, then wait for the user.
- Examples: `/say "Issue 727 needs your input on the plan"`, `/say "Issue 782 has merge conflicts"`.

### Phase 4: Summary
14. After all issues finish (or are blocked), print a table:
    - Issue | Branch | PR | Status (`merged` / `open-needs-review` / `blocked` / `failed`)
    - List any blocked/failed issues with the specific reason.

## Constraints
- **Always use worktrees** (never modify the user's main checkout).
- **Sequential by default**: only enable `--parallel` when explicitly requested.
- **Parallel cap**: max 3 concurrent issues per wave (via sub-agents).
- **Max 2 fix-review cycles** per issue, then escalate via `/say`.
- **Never force-push** without explicit user confirmation.
- **Never skip hooks** (`--no-verify`).
- A single failed/blocked issue does NOT halt the rest of the run — mark it and continue.
- No emojis, no time estimates in the summary.

## Examples
- `/issues-run 727 782 783` — sequentially do the full cycle for each issue, in the listed order (or a reordered sequence if dependencies suggest it).
- `/issues-run 727,782,783 --parallel` — run in waves, up to 3 concurrent.
- `/issues-run 727 782 --no-merge` — open PRs and pass review, but stop before merge.
- `/issues-run 727 782 783 --dry-run` — show the planned order and exit.
