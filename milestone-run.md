---
description: Execute a GitHub milestone by planning and implementing its issues in order, with parallel execution support. Use when user wants to execute a milestone, run a milestone, or implement all issues in a milestone.
---

# Milestone Run

Execute a GitHub milestone end-to-end: plan, implement, review, merge, close — fully autonomous with parallel execution.

## Phase 1: Load and Analyze

1. **Fetch milestone** — Get the milestone's open issues and its execution-plan comment, pulling only the fields used: `gh issue list --milestone "<title>" --state open --json number,title,labels,body` (plus the comment). Do not dump raw full milestone `gh api` JSON into context. If no open issues remain, skip to Phase 3 (the milestone may still need its final integration→main merge).
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

5. **Process waves sequentially** — For each issue in the wave, spawn one sub-agent that runs the *entire* per-issue pipeline (steps 7–9: pre-start check, implement, review, merge) in its own context. Up to 3 such sub-agents run in parallel per wave. The orchestrator never inlines a per-issue `/plan-ok`, `/code-review`, review-polling, or fix cycle — all of it lives inside the sub-agent, so its file reads, edits, and review-comment dumps stay out of the orchestrator's context. Each sub-agent returns ONLY a compact summary; the orchestrator must not pull the sub-agent's transcript.

   Per-issue summary schema (the sub-agent's entire return value):
   ```
   issue: <number>
   status: done | needs-review | blocked | failed
   pr_url: <url or none>
   merged: true | false
   acceptance: all-met | unmet:[<criteria that failed the spec-conformance gate>]
   blockers: [<external blockers, if any>]
   decisions_affecting_later_issues: [<short notes later issues need, e.g. "renamed config key X→Y">]
   notes: <one line, optional>
   ```
   Track wave status from these summaries: `pending` / `in-progress` / `done` / `blocked` / `failed`. The `decisions_affecting_later_issues` field is how cross-issue context survives without keeping each issue's full transcript. After each issue closes, the orchestrator appends its decisions to `DECISIONS.md` at the root of the integration worktree — an untracked local file (do NOT commit it; it must not merge into `main`), serving as a small persistent knowledge base (the environment layer). Each issue's sub-agent reads `<integration-worktree-path>/DECISIONS.md` at pre-start, so later waves see earlier waves' decisions as a file, not only as passed-in prose.

6. **Pre-wave sync** — Before starting each wave (including the first). All commands run inside the integration worktree via `git -C <integration-worktree-path>`; the project's main directory is never touched.
   - Fetch `main`: `git fetch origin main`.
   - Update integration worktree from remote: `git -C <integration-worktree-path> pull --ff-only origin <integration-branch>`.
   - Merge `main` into integration in the worktree: `git -C <integration-worktree-path> merge --no-edit origin/main`.
   - **On conflict**: halt the milestone, leave the conflicted state in place inside the integration worktree, print `MILESTONE HALTED: integration branch conflicts with main at <integration-worktree-path> — resolve and re-run /milestone-run`, comment on the milestone, exit.
   - **On clean merge**: `git -C <integration-worktree-path> push origin <integration-branch>`. Continue to step 7.
   - This means each wave's issue worktrees fork off the latest integration HEAD, which includes both prior waves' merges and any updates from `main`.

**Steps 7–9 run inside the per-issue sub-agent** (one per issue, ≤3 parallel per wave), not in the orchestrator. The sub-agent performs all of the following, then returns the step-5 summary as its only output.

7. **Pre-start check** — Before starting the issue:
   - Read `<integration-worktree-path>/DECISIONS.md` (if present) for decisions from earlier issues, so this issue's work is consistent with them.
   - Ensure the plan exists: if `todo__<issue-number>.md` is present (upfront mode), use it; if absent (lazy mode — the default), generate it now inside this sub-agent: `/make-plan --from-issue <number> --tree --base <integration-branch>`. Either way the generation and its codebase reads happen in the sub-agent's context, not the orchestrator's. Generating lazily against the current integration HEAD also means later-wave plans reflect prior waves' merged work.
   - Confirm the plan carries the issue's **acceptance criteria** (each with its verification method). If missing, regenerate the plan — the spec-conformance gate in step 9 needs them.
   - Confirm the plan's `Base:` line matches the current integration branch. If it points to `main` or a stale branch, regenerate.
   - Re-read the issue from GitHub (`gh issue view`). Compare against the plan. If the issue was materially updated since the plan was created (new requirements, changed scope, new comments with decisions), regenerate the plan with `--base <integration-branch>`.

8. **Implement** — Run `/plan-ok <number>` (it reads the base from the plan; `CC_BASE_BRANCH` is also exported as a safety net). On external blockers: print `BLOCKED: Issue #<number> — <description>`, comment on issue, mark blocked, move on. On technical failure: retry once, then mark `failed`.

9. **Verify, Review, and Merge** (issue PR → integration branch, NOT main):
   - **Spec-conformance gate (run before code review).** This checks "did we build what the spec said," which is distinct from code review's "is this good code." Both gates must pass.
     - **External signal**: run the issue's verification plan from its acceptance criteria — execute the named tests/commands and run `/verify` to exercise the actual behavior. Record pass/fail per criterion.
     - **Spec critic**: spawn one independent sub-agent whose only job is to judge the diff against the issue's acceptance criteria and invariants, returning a per-criterion verdict (met / not-met / unclear) with evidence. It does not assess code quality.
     - **Gate**: every acceptance criterion must be met (tests green and critic confirms). If any fails, fix and re-run the gate (up to 2 cycles). Do not merge with unmet criteria; if still unmet after 2 cycles, set `status: needs-review`, record the failed criteria in `acceptance: unmet:[...]`, and stop without merging (do not block the wave).
   - `/git-pre-pr --base <integration-branch>` → `/git-pr --base <integration-branch> --issues "<number>"`
   - Request `/copilot-review`, then `/code-review:code-review`
   - Wait up to 10 min for both reviews — poll inside this sub-agent and keep only the final outcome; do not surface per-minute status to the orchestrator.
   - `/gh-code-review --retry` → fix issues → `/git-add-commit-push`
   - Max 2 fix-review cycles, then escalate by setting `status: needs-review` in the summary (do not block the wave).
   - `/git-pr-merge` → `/pr-merged` → verify issue closed
   - `/git-pr-merge` automatically checks out the PR's base (the integration branch), so no `main` checkout happens here.
   - **Return** the step-5 summary as the sub-agent's entire output — nothing else.

10. **Post-close refresh** — After each issue is closed:
    - Re-fetch only issue numbers and `updatedAt` (`gh issue list --milestone "<title>" --json number,updatedAt`) to detect additions or changes cheaply. Pull a full body only for an issue that is new or whose `updatedAt` moved — do not re-dump every body.
    - If new issues found: incorporate them into the execution order and determine wave placement from dependencies. Plan generation for new issues follows the run's mode — lazy by default (their sub-agents generate plans at pre-start); only generate now if running upfront.
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
- Max 3 parallel issues, max 2 fix-review cycles per issue, max 2 spec-conformance-gate cycles per issue
- The spec-conformance gate (tests + `/verify` + spec critic against the issue's acceptance criteria) must pass before merge. It is separate from code review and runs first. No merge with unmet acceptance criteria.
- Always use worktrees. Never force-push without confirmation.
- Blocked issues don't block the pipeline. Single failure doesn't stop the milestone.
- Failed dependencies mark dependents as `blocked`.
- Network/auth errors: retry once, then escalate.
- No emojis, no time estimates.
- Issue PRs target the milestone integration branch, never `main`. The integration→main merge requires explicit user confirmation and happens only after all issues are closed.
- `main`→integration auto-merge runs at every wave boundary. On conflict, halt the milestone — do not attempt automatic resolution.
- `CC_BASE_BRANCH` must be exported for the entire run so sub-commands inherit the integration branch as their base.
- The integration branch lives in a dedicated worktree (`.trees/milestone-<n>-<slug>` by default). All milestone-level git operations use `git -C <integration-worktree-path>`. The project's main directory is never checked out to the integration branch and must be left untouched by milestone-run.
