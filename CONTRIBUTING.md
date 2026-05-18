# Contributing to claude-shipyard

Thanks for considering a contribution. These commands exist because they get used every day — the bar for adding or changing one is "does it earn its keep across multiple projects."

## Ground rules

- **One file per command.** The filename (without `.md`) becomes the slash-command name.
- **Front-matter is required.** Every command starts with:
  ```yaml
  ---
  description: One-sentence summary used by Claude to decide when to invoke the command.
  ---
  ```
  The `description` is what Claude reads to route a request. Be specific — include trigger words ("when the user asks…").
- **Reuse, don't reinvent.** If your new command needs to commit + push, call `/git-add-commit-push`, don't re-implement it. The same goes for `/git-pr`, `/git-pre-pr`, `/copilot-review`, etc.
- **Document the arguments.** Use the `## Arguments (from {{ARGS}})` section. List every flag with a short description and a default.
- **No personal references.** No hardcoded paths, emails, usernames, or project-specific assumptions. Use `<owner>/<repo>` style placeholders.
- **No emojis** in command prompts unless the user explicitly asks for them at runtime.
- **No `Signed-off-by` or AI-attribution trailers** in any auto-generated commit or PR body.

## Adding a new command

1. Pick a name. Verb-led, kebab-case, namespaced where it makes sense (`git-foo`, `milestone-foo`).
2. Copy an existing command of similar shape as a starting point — `/git-pre-pr` is a good model for "do checks + report," `/issues-run` for "orchestrate other commands."
3. Open a PR. In the description, explain:
   - The problem the command solves.
   - When you'd invoke it (so the `description` is discoverable).
   - Which existing commands it composes (if any).

## Changing an existing command

- Behavior changes that affect callers (different defaults, removed flags) deserve a heads-up in the PR body — several commands call each other.
- Skill-prompt edits should preserve the existing section structure (`## Behavior`, `## Arguments`, `## Constraints`, etc.) so the file stays easy to scan.

## Testing changes

There's no test harness. Manual verification is the bar:

1. Drop the modified file into your `~/.claude/commands/`.
2. Run the command on a throwaway repo or worktree.
3. Confirm the happy path and at least one obvious failure mode.

Note the test in your PR description ("ran `/issues-run 1 2 --dry-run` against a scratch repo, got X").

## Reporting issues

GitHub issues are fine for bug reports and feature ideas. Include:
- The command you ran and the arguments.
- What you expected.
- What happened (paste relevant output).
- Your Claude Code version and OS.
