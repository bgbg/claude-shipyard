---
description: Review user-edited plan for contradictions and validate coherence
---

# Plan Updated

Review user's changes to the plan, check for contradictions, validate coherence, confirm readiness.

## Behavior

### 1) Determine plan source
- `--file <path>`: explicit plan file
- `--issue <number|url>`: GitHub issue (check for local `todo__<issue-number>.md` first)
- `<argument>`: smart detection — numeric = issue number, contains "github.com" = issue URL, otherwise = file path
- No arguments: auto-detect from `todo__*.md` files modified in last 24h

### 2) Load plan content
- **File**: Read directly. Extract issue number from `todo__<N>.md` filename.
- **Issue**: Use local todo file if exists, otherwise fetch from issue comments (newest-first).
- **No args**: If single recent file found, use it. If multiple, use context to select and verify with user.

### 3) Analyze for contradictions
- **User edits always take precedence** — update contradicting AI-generated parts silently when intent is clear.
- If unresolvable contradictions: show with line references, ask user to clarify. Don't proceed until resolved.

### 4) Validate completeness
- Verify open questions with multiple options have user's indicated preference
- Check answers are reflected in Approach and Steps sections
- Verify steps align with stated approach and dependencies are noted

### 5) Unify language and structure
Minor edits for readability and formatting consistency.

### 6) Final coherence pass
Read as a reviewer: Does it make sense? Are steps achievable? Does approach address answered questions? Is scope clear?

### 7) Confirm readiness
- **Ready**: Summarize key changes (2-3 bullets), confirm no contradictions. With `--proceed`: auto-invoke `/plan-ok`. Without: state "Run `/plan-ok` to begin."
- **Issues remain**: List by severity (Critical/Major/Minor) with location, description, suggested fix. Never invoke `/plan-ok` if issues remain.

## Arguments (from {{ARGS}})
- `--file <path>`: Plan file path
- `--issue <number|url>`: GitHub issue
- `--proceed`: Auto-run `/plan-ok` when ready (default: false)
- `<argument>`: Positional with smart detection (numeric/URL/path)

## Notes
- Focus on contradictions and coherence, not style perfection
- Only flag issues that could cause implementation problems
- If plan is in good shape, confirm quickly
- No emojis, no time estimates
