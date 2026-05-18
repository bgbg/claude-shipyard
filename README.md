# claude-shipyard

A collection of [Claude Code](https://docs.claude.com/en/docs/claude-code) slash commands for shipping software through GitHub — issues, plans, branches, PRs, reviews, merges, milestones.

These are personal commands that have evolved over months of daily use. They lean heavily on `gh`, git worktrees, and other Claude Code commands as composable building blocks.

## What's in here

### Issue → branch → plan
- **`/git-make-issue`** — Generate a clear GitHub issue title + description, then create it.
- **`/git-work-on-issue`** — Create a properly-named branch and worktree for an issue, label it `work-in-progress`, and generate a plan.
- **`/make-plan`** — Generate an implementation plan from a GitHub issue or general task. Writes `todo__<n>.md`.
- **`/plan-ok`** — Plan is approved; start implementing it.
- **`/plan-updated`** — Re-validate a user-edited plan for contradictions.

### Code review and PRs
- **`/git-pre-pr`** — Comprehensive self-review before opening a PR (tests, secrets scan, exception-handling check, maintainability, missing tests, etc.).
- **`/git-pr`** — Create a well-formed PR with auto-generated title and body.
- **`/copilot-review`** — Request a GitHub Copilot review on the current PR.
- **`/gh-code-review`** — Read and analyze code-review comments, grouped by severity.
- **`/git-add-commit-push`** — Stage, commit (with an auto-generated message), and push.
- **`/git-pr-merge`** — Merge the current PR and clean up branches/worktrees.
- **`/pr-merged`** — Post-merge cleanup.

### Multi-issue orchestration
- **`/issues-run`** — Run a full plan→implement→review→merge cycle over a list of issues. Sequential by default, `--parallel` for waves. Plans first (where humans intervene most), then execute.
- **`/milestone-plan`** — Turn a brainstorm or design doc into a GitHub milestone with issues and a dedicated integration branch + worktree.
- **`/milestone-run`** — Execute a milestone end-to-end. Issue PRs target the milestone integration branch, never `main`, until everything's ready.
- **`/brainstorm`** — Iterative brainstorm for large milestones, features, or architectural decisions.

### Misc
- **`/dataviz`** — Create or revise a publication-quality figure following Tufte/Few/Doumont principles (matplotlib/seaborn defaults baked in).
- **`/marp-presentation`** — Build a Marp slide deck from scratch via guided Q&A.
- **`/whats-the-status`** — Project status: branches vs. todo files.
- **`/say`** — Speak text aloud via macOS `say`.

## Installation

Claude Code scans `~/.claude/commands/` for `.md` files and turns each one into a slash command. The cleanest way to install is to clone the repo somewhere outside `~/.claude/commands` and copy (or symlink) only the command files:

```bash
git clone https://github.com/bgbg/claude-shipyard.git ~/src/claude-shipyard

# Copy all commands (skip README/LICENSE/CONTRIBUTING — those aren't commands)
cd ~/src/claude-shipyard
for f in *.md; do
  case "$f" in README.md|CONTRIBUTING.md) continue;; esac
  cp "$f" ~/.claude/commands/
done

# Or pick the ones you want
cp ~/src/claude-shipyard/git-pre-pr.md ~/.claude/commands/
```

> **Heads up:** if you clone the repo directly *into* `~/.claude/commands/`, the harness will pick up `README.md` and `CONTRIBUTING.md` as bogus `/README` and `/CONTRIBUTING` slash commands. Use the copy/symlink approach above instead.

Restart Claude Code and the commands appear as `/git-pre-pr`, `/issues-run`, etc.

## Design notes

- **Commands compose.** Big commands like `/issues-run` and `/milestone-run` are thin orchestrators — they delegate to the smaller commands. If you fix a bug in `/git-pre-pr`, every workflow benefits.
- **Worktrees by default.** Long-running work happens in `.trees/<branch>` so the user's main checkout is never disturbed.
- **`CC_BASE_BRANCH` env var** flows through the chain so milestone integration branches (or any non-`main` base) are honored end-to-end.
- **Humans in the loop.** Commands `/say` when they need attention, and Open Questions in plans are surfaced explicitly rather than guessed.
- **Bias for `gh`.** `gh` over the GitHub REST API; assume auth is configured.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md). PRs welcome — especially for commands that generalize well across projects.

## License

[MIT](LICENSE).
