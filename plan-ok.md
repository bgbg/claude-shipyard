---
description: Plan is approved and ready to start work
---

# Plan OK

Start implementing the plan, adhering to it and the user's instructions.

## Behavior

### 1) Determine plan source
- `--file <path>`: explicit plan file
- `--issue <number|url>`: GitHub issue
- `<argument>`: smart detection — numeric = issue number, contains "github.com" = issue URL, otherwise = file path
- No arguments: ask user

### 2) Fetch plan content
- **File (preferred)**: Read locally. Extract issue number from `todo__<N>.md` filename. Do NOT fetch from GitHub.
- **Issue**: Use local `todo__<issue-number>.md` if exists. Otherwise fetch from issue comments (newest-first, look for plan sections). Abort if no plan found.

### 3) Parse and validate
- **Branch strategy**: Extract from "## Branch Strategy" section. Read `Branch:`, `Worktree:`, and `Base:` lines. If missing and not in Open Questions, abort: "Cannot proceed - plan must specify branch name and worktree preference"
- **Base branch resolution**: Use the plan's `Base:` line if present. If missing, fall back to `CC_BASE_BRANCH` env var, then to the repo default branch. Do not silently default to `main` if the plan was generated with a non-default base.
- **Open questions**: If "No questions", proceed. If all single-option, proceed automatically. If multi-option unanswered, present to user and wait.
- **Contradictions**: Answers take precedence over plan text — update plan silently. If answers contradict each other, ask user.

### 4) Post plan to GitHub (in parallel)
- `gh issue edit <issue-number> --add-label "work-in-progress" --remove-label "planning"`
- `gh issue comment <issue-number> --body "<plan>"` (heredoc, no signature lines)

### 5) Setup branch and worktree
Use the resolved base branch from step 3.

**Worktree (default)**: Check `git worktree list`. If exists for branch, switch to it. Otherwise create at `.trees/<branch-name>`, forked from the resolved base: `git worktree add <path> -b <branch-name> origin/<base>`. Fetch the base first if needed: `git fetch origin <base>`.

**No worktree**: Checkout the resolved base, fast-forward, then create the new branch off it. If the branch already exists, just check it out.

### 6) Work
- Implement step by step. Test as you go, fix errors immediately.
- Periodically commit and push (after completing steps or significant milestones).
- Ask user if tasks require their intervention.

### 7) Review step
- `/git-pre-pr --base <resolved-base>` → `/git-pr --base <resolved-base>` (omit `--base` if it equals the repo default)
- If `--no-code-review` is NOT set:
  1. Request `/copilot-review` first (remote), then run `/code-review:code-review` (local)
  2. Wait up to 10 minutes, checking every minute. Stop when BOTH reviews are ready.
  3. `/gh-code-review --retry` to read comments
  4. Fix all problems, commit and push via `/git-add-commit-push`
  5. If reviews had **critical** or **serious** problems, repeat steps 1–4 once more (max 2 rounds)

## Arguments (from {{ARGS}})
- `--file <path>`: Plan file path
- `--issue <number|url>`: GitHub issue
- `<argument>`: Positional with smart detection (numeric/URL/path)
- `--no-code-review`: Skip PR creation and code review
