---
description: Work on a list of GitHub issues end-to-end (plan, implement, review; merge unless the base is the repo default branch). Claims all issues as work-in-progress immediately, then auto-decides per issue between a shared context and isolated sub-agents (`--shared`/`--isolated` to force); `--parallel` for concurrency. Use when the user lists multiple issue numbers and wants the full cycle for each.
---

# Issues Run

Execute multiple GitHub issues end-to-end: for each issue, run the full cycle — plan, implement, code review, merge — using a dedicated branch and worktree per issue. Issues run either inline in one shared context or each in its own isolated sub-agent; by default the command decides per issue after planning (see Phase 1.5). Reuses existing skills; this command is just the orchestrator.

## Arguments (from $ARGUMENTS)
- `<issues>`: One or more issue numbers, space- or comma-separated (e.g., `727 782 783` or `727,782,783`). Required.
- `--auto` (alias `--consider-isolated`): **default**. Decide per issue — after planning — whether to run it in its own isolated sub-agent or inline in the shared orchestrator context (see Phase 1.5).
- `--shared`: force every issue to run inline in one shared context (no per-issue sub-agents). Good for batches of small, cheap, often-unrelated chores.
- `--isolated`: force every issue into its own sub-agent so the orchestrator stays lean (the model `/milestone-run` uses).
- `--parallel`: run isolated issues concurrently (max 3 at a time). Applies only to issues that run isolated — all of them under `--isolated`, the isolated subset under `--auto`; ignored for issues running shared/inline.
- `--no-reorder`: Keep the user-specified order. Default: allow reordering when dependencies suggest a better order.
- `--base <branch>`: Base branch for all issues. Default: `CC_BASE_BRANCH` env var, else repo default branch.
- `--merge`: Also merge PRs whose base is the repo default branch. Default: a PR into the default branch is left open for the user (`/git-pr-merge` from the issue's worktree merges and cleans up); a PR into any other base (e.g. a milestone integration branch) merges automatically.
- `--no-merge`: Never merge, whatever the base.
- `--dry-run`: Print the planned execution order and exit without doing work.

## Behavior

### Phase 0: Claim, validate, and order
1. Parse the issue list. Abort if any value is not a positive integer.
2. **Claim the issues immediately — before anything else** (skip only if `--dry-run`): in parallel, `gh issue edit <n> --add-label "work-in-progress"` for every parsed issue number. This is the run's very first GitHub action, so no other person or process can pick up an issue while this run prepares it. Non-blocking: log any failures but continue.
3. Fetch each issue in parallel: `gh issue view <n> --json number,title,state,labels,body,assignees`.
   - Abort if any issue is closed or missing. On abort, remove the `work-in-progress` label this run just added in step 2, so no stale claim is left behind.
   - **Collision check**: if a fetched issue is already assigned to someone else, warn of a possible collision (in chat, and `/say` if the user has stepped away) and ask whether to continue with it or drop it.
4. **Determine order**:
   - Default order = user-specified order.
   - Unless `--no-reorder`: consider whether a different order would be better. Signals to weigh:
     - **Dependencies**: scan each issue body for `Blocked by #N` / `Depends on #N` references among the listed issues.
     - **Other considerations**: shared files/areas (do related issues together), risk (land low-risk first), size (quick wins first), or any ordering hint in the issues themselves.
     - If a better order emerges, propose it with a one-line rationale and ask for confirmation.
   - For isolated issues run with `--parallel`: group into waves by dependency (issues with no unmet deps among the list go in wave 1, etc.). Max 3 per wave.
5. Print the resolved plan: ordered list (or waves) + base branch + execution mode. If `--dry-run`, exit here (nothing was claimed; no work done).

### Phase 1: Plan all issues (inline, regardless of execution mode)
Plans require the most user intervention (open questions, scope clarifications), so always do this serially first — get all the human-in-the-loop work out of the way before any long-running implementation starts.

6. For each issue **in order**:
   - Run `/git-work-on-issue <issue> --base <base>` — creates the branch, worktree (its default), and the plan via `/make-plan`, then stops. Do not pass it `--auto` or `--skip-plan` (its `--auto` is unrelated to this command's `--auto`); Phase 2 runs `/plan-ok`.
   - If `/make-plan` raises Open Questions that need answers, resolve them with the user now. Use `/say` to alert the user if they have stepped away.
   - Confirm `todo__<issue>.md` exists in the root of the main checkout, its `Base:` line matches the resolved base, and it carries the issue's **Acceptance Criteria** (each with a verification method). If criteria are missing, regenerate the plan — the spec-conformance gate in Phase 2 needs them.
7. After all plans are ready, print a one-line summary per issue (branch, worktree path, plan file).

### Phase 1.5: Decide isolation per issue
Skip under `--shared` (everything inline) or `--isolated` (everything isolated). Under `--auto` (default), classify each issue now that its plan exists — the plan is the richest signal.

8. For each issue, choose **isolated** (own sub-agent) or **shared** (inline):
   - **isolated** when the plan has many steps, touches many files, has a large blast radius, carries Open Questions, or the issue is labeled `feature`/`enhancement`/`refactor`/`epic`; also lean isolated when the overall batch is large (protect the orchestrator's context).
   - **shared** when the plan is tiny (1–2 steps, ~1 file) and the task is a chore/doc/typo/trivial fix.
   - **Tie-break**: when unsure, go by plan size: **shared** for a small plan, **isolated** otherwise. A sub-agent adds cost; use it when the issue's reads and edits would crowd the orchestrator's context.
9. Print a decision table (Issue | mode | one-line rationale) and proceed. The user overrides by re-running with `--shared` or `--isolated`.

### Phase 2: Develop → Verify → PR → Review → Merge → Cleanup
Run the per-issue cycle below for each issue. **How** it runs depends on the issue's mode from Phase 1.5:
- **Isolated**: one sub-agent runs the entire cycle (steps 10–15) in its own context and returns ONLY the compact summary below; the orchestrator never pulls its transcript. Isolated issues run sequentially, or in parallel waves of ≤3 under `--parallel`, respecting dependency order. Brief each sub-agent with: issue number, base branch, worktree path, plan file path, the summary schema, and the escalation rule (it cannot ask the user, so it reports `blocked` or `needs-review`).
- **Shared**: the orchestrator runs the cycle inline, one issue after another, in the shared context.

Isolated-issue summary schema (the sub-agent's entire return value):
```
issue: <number>
status: done | needs-review | blocked | failed
pr_url: <url or none>
merged: true | false
acceptance: all-met | unmet:[<criteria that failed the spec-conformance gate>]
blockers: [<external blockers, if any>]
notes: <one line, optional>
```

Per-issue cycle:

10. **Implement**: `/plan-ok <issue> --no-pr` — runs from inside the worktree. The plan's `Base:` line drives the branch base.
11. **Pre-PR check + spec-conformance gate**: `/git-pre-pr` (with `--base <base>` if non-default). Its spec-conformance gate (step 2) verifies the build against the plan's acceptance criteria (tests + `/run` + spec critic), distinct from code review's "is this good code." If the verdict is `fail` or `incomplete`, fix and re-run (up to 2 cycles). Do not open the PR or merge with unmet criteria; if still unmet after 2 cycles, escalate via `/say`, mark the issue `open-needs-review`, and move on (do not halt the run).
12. **Open PR**: `/git-pr --issues "<issue>"` (with `--base <base>` if non-default)
13. **Review**: `/git-pr` in step 12 runs `/code-review high --fix` and commits the fixes, max 2 rounds. Do not run another review.
14. **Escalate**: if critical or serious review problems remain after those 2 rounds, escalate via `/say`.
15. **Merge and cleanup**: only when the base is not the repo default branch, or `--merge` was given; never under `--no-merge`. `/git-pr-merge --no-fix` (the review fixes already happened in step 13). Verify the issue is closed and the worktree is removed. Otherwise leave the PR open with status `open` and keep the worktree — the user merges with `/git-pr-merge`.

### Phase 3: Human intervention (applies throughout)
- Whenever the run hits a blocker that needs the user (unanswered plan questions, failing tests after retries, unresolved review feedback, merge conflicts, auth failures), call `/say` with a short message naming the issue and the blocker. Mark the issue and continue with the issues that do not depend on it. Wait for the user only when nothing else can proceed (plan questions in Phase 1 are the usual case).
- Examples: `/say "Issue 727 needs your input on the plan"`, `/say "Issue 782 has merge conflicts"`.

### Phase 4: Summary
16. After all issues finish (or are blocked), print a table:
    - Issue | Mode (`isolated`/`shared`) | Branch | PR | Status (`merged` / `open` / `open-needs-review` / `blocked` / `failed`)
    - For `open` PRs, print the `/git-pr-merge` command per issue (with its worktree path).
    - List any blocked/failed issues with the specific reason. For `open-needs-review` due to unmet acceptance criteria, name the failed criteria.

## Constraints
- **Claim first**: mark every issue `work-in-progress` before any other GitHub action (Phase 0 step 2), so the run owns the issues immediately. Skip only under `--dry-run`.
- **Always use worktrees** (never modify the user's main checkout).
- **Execution mode**: `--auto` (default) decides isolated-vs-shared per issue after planning; `--shared` forces all inline; `--isolated` forces all into sub-agents. `--shared` and `--isolated` are mutually exclusive (error if both given).
- **Concurrency**: `--parallel` (max 3 at a time) applies only to isolated issues; it is ignored for issues running shared/inline.
- **Isolation tie-break**: under `--auto`, when unsure, go by plan size (Phase 1.5).
- **Max 2 fix-review cycles** per issue, then escalate via `/say`.
- **Spec-conformance gate** runs inside `/git-pre-pr` (verifies the build against the issue's acceptance criteria: tests + `/run` + spec critic). Its verdict must be `pass` before opening the PR/merge; `fail`/`incomplete` → fix and re-run, max 2 cycles, then escalate via `/say` and mark `open-needs-review`. No merge with unmet criteria.
- **No unattended merge into the default branch**: PRs into the repo default branch stay open for the user unless `--merge`. PRs into other bases merge automatically.
- A single failed/blocked issue does NOT halt the rest of the run — mark it and continue.
- No emojis, no time estimates in the summary.

## Examples
- `/issues-run 727 782 783` — auto mode: claim all three immediately, plan, then decide per issue whether to isolate or run inline, in dependency-aware order.
- `/issues-run 727,782,783 --shared` — handle all three inline in one shared context (good for small, unrelated chores).
- `/issues-run 727 782 783 --isolated --parallel` — each issue in its own sub-agent, up to 3 concurrent.
- `/issues-run 727 782 --merge` — also merge the PRs into the default branch instead of leaving them for `/git-pr-merge`.
- `/issues-run 727 782 783 --dry-run` — claim nothing; show the planned order and modes, then exit.
