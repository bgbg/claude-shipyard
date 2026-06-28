---
description: Generate implementation plan from GitHub issue or general task
---

# Plan work on a task

Create a concise, implementation-ready plan. DON'T change any existing code.

## Branch Strategy Detection
Before generating, determine branch strategy:
- Branch name: user-specified `--branch <name>`, or auto-generate as `[feat|develop|fix|chore]/<issue_number>-short-description`
- Worktree: default ON unless `--no-tree` specified
- Base branch: resolve in priority order: explicit `--base <branch>` argument → `CC_BASE_BRANCH` environment variable → repo default branch (typically `main`). Always record the resolved base in the plan.
- If branch/worktree preference is clear: include "Branch Strategy" section in plan
- If unclear: omit section, add Open Question Q0 asking about it with options and implications

## Issue Mode (`--from-issue <number|url>` or positional number)
1. **Fetch in parallel**: issue data (`gh issue view <issue> --json number,title,state,body,comments,repository`) and add "planning" label (non-blocking)
2. Verify issue is open. Abort if closed.
3. **Check for existing plan** (from fetched comments, newest-first):
   - **Found**: Copy to `todo__<issue-number>.md`, append any comments on open questions. Do not regenerate. Exit.
   - **Not found**: Check for existing `todo__<issue-number>.md`. If exists, ask user whether to regenerate or keep.
4. Only if no existing plan: analyze issue content and generate plan
5. **Write** `todo__<issue-number>.md` (absolute path, in `pwd` — NOT in worktree)

## General Mode
1. First argument = output file path
2. Generate plan using entire project as context
3. **Write** specified output file (absolute path)

## Post-Generation
If steps include sub-issues, offer to create them via `gh issue create` and update plan with actual numbers.

## Plan Structure

Required sections:
1. **Title**: one-line goal. Good: "Multi-worker deployment (#311)". Bad: "Implementation Plan: Issue #311"
2. **Branch Strategy** (if determined; otherwise capture as Q0). Must include three lines: `Branch: <name>`, `Worktree: <yes|no>`, `Base: <branch>`. The `Base:` line is required so `/plan-ok` knows what to fork the worktree from.
3. **Issue Summary** (issue mode only)
4. **Assumptions**: key assumptions affecting scope/design
5. **Open Questions**: Q0 for branch strategy if needed, then Q1, Q2... Each with up to 3 options (A/B/C) with 1-2 sentence implications. "No questions" if none.
6. **Approach**: rationale, alternatives briefly noted
7. **Steps**: numbered, outcome-focused. For complex tasks, break into sub-issues.
8. **Acceptance Criteria** (required): a checklist where every item is a falsifiable statement that names its verification method — the test to run, the command/`/verify` steps to execute, or the manual check. In issue mode, carry these down from the issue's spec/acceptance criteria (set by `/milestone-plan`); if the issue has none, write them and note the gap. These are what `/milestone-run` verifies the PR against before merge — not optional.

Optional (add only if they reduce ambiguity): Scope/Non-Goals, Requirements/Constraints, Risks/Mitigations, Dependencies, Deliverables, Rollout/Backout.

## Visual (optional)

Add a `## Visual` section **only when a picture genuinely clarifies the plan** — multi-module work, non-trivial flows, schema changes, or new components with non-obvious relationships. Skip it for single-file changes, simple bug fixes, or anything you could describe in one sentence. Never pad a plan with filler diagrams.

Use one (rarely two) of the following, in this order of preference:

- **Mermaid diagrams** — preferred because GitHub renders them natively in issues, PRs, and Markdown. Pick the type that matches the actual question the reader has:
  - `graph TD` / `graph LR` for component or module relationships
  - `sequenceDiagram` for an interaction across services, processes, or actors
  - `erDiagram` or `classDiagram` for a data model
  - `flowchart` for branching logic that's hard to describe in prose
- **Markdown file tree** — nested bullets showing new/touched paths, grouped by top-level directory. Use when the change is primarily structural (new module, large refactor).

Rules:
- Keep diagrams under ~15 nodes. If you need more, split into two diagrams or drop it.
- Label nodes with the actual symbol/file/component name from the codebase — no placeholders like "Service A".
- Mark anything that doesn't exist yet (e.g. `NewWorker[NewWorker (new)]`) so reviewers can tell at a glance what's the delta.
- One diagram, not a gallery. The goal is the fastest possible read, not a documentation set.

## Arguments (from {{ARGS}})
- `<output-file>`: Output path (general mode)
- `--from-issue <number|url>` or `<number>`: GitHub issue
- `--tree` / `--no-tree`: Worktree preference (default: tree)
- `--branch <name>`: Custom branch name
- `--base <branch>`: Base branch to fork from. Resolution order: this flag → `CC_BASE_BRANCH` env var → repo default branch.

## Style
- Expert audience, crisp, skimmable. Bullets over paragraphs. Backticks for code.
- Numbered flat lists over nested bullets. No time estimates or emojis.
- LLM-optimized: deterministic, explicit, no pronouns. Consistent terminology.
- Sub-issues: basic markdown only for `gh issue create` compatibility.
