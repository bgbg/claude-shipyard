---
description: Stage changes, generate commit message, commit, and push
---

# Git Add Commit Push

Stage changes, generate a commit message, commit, and push for the current project.

## Behavior
1) Handle untracked files:
   - If untracked files exist and `--all` is not set, ask to include them or ignore.
   - If `--no-confirm` is set, default to adding untracked files.

2) Stage changes:
   - If including untracked: `git add -A`
   - Otherwise: `git add -u`

3) Check branch name for issue reference:
   - Skip this step if `--no-issue` is set
   - If branch name contains a numbered issue pattern (e.g., `issue-123`, `fix/issue-42`, `123-fix-bug`), extract the issue number
   - **Auto-include** `Related to #<NUMBER>` in the commit message without asking the user
   - If `--closes` flag is set, use `Closes #<NUMBER>` instead of `Related to #<NUMBER>`
   - Do NOT prompt the user about issue references — just include them silently

4) Generate commit message:
   - If `--message` provided: use it exactly
   - Default (simple mode): `<type>: <brief description>` using file paths and their names. If total diff is shorter than 1000 characters, use that to generate a more detailed message, unless --fast is set.
   - If `--detailed`: full conventional commit with body and analysis
   - Append issue reference if user confirmed in step 3

5) Commit and push:
   - Commit: `git commit -m "<message>"`
   - Push to upstream or set upstream if needed

## Arguments (from {{ARGS}})
- `--all`: Include untracked files
- `--message <msg>`: Use provided commit message
- `--fast`: Generate a simple commit message (faster)
- `--simple`: If total diff is shorter than 1000 characters, use that to generate a more detailed message
- `--detailed`: Generate detailed conventional commit (slower)
- `--confirm` and `--no-confirm`: whether to skip confirmation prompts (DEFAULT: --no-confirm)
- `--no-issue`: Skip checking branch name for issue references
- `--closes`: Use `Closes #<NUMBER>` instead of `Related to #<NUMBER>` for issue references

## Session Behavior
After completing the git add-commit-push operation, for the rest of the session, do not add, commit, or push any changes unless explicitly asked to do so by the user.
