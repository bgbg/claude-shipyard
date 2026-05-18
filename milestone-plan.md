---
description: Create a GitHub milestone with issues from a brainstorm or design document. Use when user wants to break down a design into a milestone with trackable issues, or when brainstorm ends and user wants to proceed.
---

# Milestone Plan

Take a design document (from `/brainstorm`, a markdown file, or free-form description) and create a GitHub milestone with well-scoped issues, execution order, and dependency tracking.

## Behavior

### Phase 1: Ingest Design

1. **Determine input**: `--file <path>` for markdown file, `<text>` for free-form, or auto-detect from brainstorm session. Accept any markdown file.
2. **Analyze design and codebase** — Read design document, CLAUDE.md, and relevant code. Verify any referenced existing code is accurate.

### Phase 2: Break Down into Issues

3. **Decompose into issues** — Each issue should be sized for a single PR (bot-reviewed, so can be meatier than human-atomic). Balance granularity vs chunk size. Each issue needs:
   - Clear title, description with context/scope/acceptance criteria
   - Dependencies on other issues (`Blocked by #N` / `Depends on #N`)
   - `blocked:user-action` label if requiring user action (API keys, manual setup, etc.)

4. **Determine execution order** — Group into waves (max 3 parallel per wave). Schedule `blocked:user-action` issues early so user has time to resolve them.

5. **Present plan to user** — Show waves, dependencies, blocked issues. Get confirmation before creating anything.

### Phase 3: Create on GitHub

6. **Create milestone**: `gh api repos/{owner}/{repo}/milestones -f title="<title>" -f description="<description>"`. Capture the milestone number for the next step.
7. **Create integration branch and worktree** — All milestone PRs target this branch instead of `main`, so partial milestone work cannot land on `main` until the milestone closes. The branch must always live in a dedicated worktree so the project's main directory is never touched by milestone operations.
   - Derive slug from milestone title: lowercase, ASCII, hyphenate, strip non-alphanumerics, cap at 40 characters, trim trailing hyphens.
   - Branch name: `milestone/<milestone-number>-<slug>`.
   - Worktree path: `.trees/milestone-<milestone-number>-<slug>`.
   - Create branch and worktree off `main`, then push:
     - `git fetch origin main`
     - `git worktree add .trees/milestone-<milestone-number>-<slug> -b <branch-name> origin/main`
     - `git -C .trees/milestone-<milestone-number>-<slug> push -u origin <branch-name>`
   - **If the branch already exists** (locally or remotely): do NOT abort. Check whether a worktree exists for it via `git worktree list`. If a worktree exists, reuse it. If not, create one at the canonical path: `git worktree add .trees/milestone-<milestone-number>-<slug> <branch-name>`. Then continue.
8. **Create issues** — Create dependency-free issues first so dependents can reference their numbers. Assign to milestone, add labels.
9. **Post execution plan comment** on milestone using this format (parseable by `/milestone-run`):
   ```markdown
   ## Execution Plan
   Integration branch: `milestone/<number>-<slug>`
   Integration worktree: `.trees/milestone-<number>-<slug>`
   ### Wave 1
   - #<number> — <title>
   ### Wave 2
   - #<number> — <title> (depends on #<dep1>, #<dep2>)
   ### Blocked: User Action Required
   - #<number> — <title>: <what user needs to do>
   ```
   The `Integration branch:` and `Integration worktree:` lines are required — `/milestone-run` parses them to know what branch to fork issue branches from, where to run merges, and what to merge issue PRs into.
10. **Generate plans for all issues** — Run `/make-plan --from-issue <number> --tree --base <integration-branch>` for each issue in execution order. The `--base` flag is critical: each issue's worktree must fork from the integration branch, not `main`. This creates `todo__<issue-number>.md` files with `Base: <integration-branch>` recorded in the Branch Strategy section. Print the list of generated files so the user can review them.
11. **Output** — Print milestone URL, integration branch name and URL, integration worktree path, issue URLs, execution order, and todo file list. Ask if user wants to run `/milestone-run`.

## Arguments (from {{ARGS}})
- `--file <path>`: Path to design/brainstorm markdown file
- `<text>`: Free-form design description
- No arguments: auto-detect from brainstorm session or ask user

## Constraints
- Never create issues without user confirmation
- Dependencies must be explicit in issue bodies
- Max 3 parallel issues per wave
- No emojis, no time estimates. Backticks for code elements.
- Issue titles: descriptive, no "Step N:" prefixes. Bodies: concise, acceptance criteria as checklist.
- Integration branch is mandatory. Issue PRs target the integration branch, never `main`. The final integration→main merge happens in `/milestone-run` after all issues are closed.
- Integration branch must live in a dedicated worktree at `.trees/milestone-<number>-<slug>`. The project's main directory must never be checked out to the integration branch — all milestone operations run inside the worktree.
