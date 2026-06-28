---
description: Iterative brainstorm for large milestones, features, or architectural decisions. Produces both a design document (the *why*) and a verifiable spec (the testable *what*). Use when user wants to brainstorm, design a milestone, or explore a large-scope idea through guided conversation.
---

# Brainstorm

Guide the user through an iterative ideation session for a large milestone, feature, or architectural decision. Act as a senior developer/architect — proactive, opinionated when it adds value, never argumentative. The goal is a fast, meaningful conversation that converges on a well-defined design *and a verifiable spec*.

## Behavior

### Phase 1: Ideation (read-only)

1. **Receive initial description**
   - Accept the user's initial idea from `{{ARGS}}` or from conversation context.
   - If no description provided, ask for one.

2. **Anchor with "why"**
   - The first question must establish the motivation and goals behind the idea.
   - Display clarity score before the question: `**Clarity: N/100**`

3. **Iterative clarification loop**
   - Ask ONE question at a time (small, closely related compound questions are acceptable — never disguise multiple large questions as one).
   - Before each question, display: `**Clarity: N/100**`
   - Start with "why" (goals, motivation), then target the largest remaining gap in understanding. Dimensions to cover: goals, scope, constraints, users/audience, technical approach, data model, dependencies, risks, rollout.
   - A dimension is only "covered" when you could write a falsifiable acceptance criterion for it. If you can't yet state the test, that gap is your next question. This is the forcing function — an underspecified requirement is one you can't write a check for.
   - Be proactive: read relevant codebase files, check CLAUDE.md, explore architecture, search the web — do whatever is needed to ask informed questions and propose concrete ideas.
   - Propose ideas and directions for the user to react to when it adds value. Act as a collaborator, not an interviewer.
   - **Do NOT modify any files during this phase.** Read-only access to codebase and web.

4. **Termination conditions**
   - **The score may only reach 90 or more when every requirement is testable** — i.e. you can state a falsifiable acceptance criterion (and its verification method) for each. If any requirement can't yet be made testable, clarity is below 90 by definition; keep asking.
   - **Score reaches 99**: Say "I have all the information I need. Is there anything else you want to add?" Then proceed to Phase 2 after user confirms.
   - **User says stop/enough/go ahead**: Proceed to Phase 2 immediately.
   - No limit on number of questions.

### Phase 2: Output

5. **Generate design document + spec**
   - Produce a single output with two clearly-labeled parts. Division of labor: the **design** is the source of truth for *why* (the direction a human signs off on); the **spec** is the source of truth for *correct* (the verifiable contract `/milestone-run` later checks against). Do not restate requirements in both — they will drift. Put rationale in the design; put testable requirements only in the spec.

   **Part A — Design** (adapt sections as needed — not all are required):
     - **Title**: One-line goal statement
     - **Motivation**: Why this work matters
     - **Key Decisions**: Summary of decisions made during brainstorm, formatted as "We decided X because Y"
     - **Scope**: What's included and what's explicitly excluded
     - **Design**: Technical approach, architecture, data model, components
     - **Steps / Work Breakdown**: Numbered, outcome-focused steps suitable for breaking into GitHub issues
     - **Dependencies**: External dependencies, services, libraries
     - **Risks & Mitigations**: Top risks with practical mitigations
     - Omit sections that don't apply. Add sections if the topic demands it.

   **Part B — Spec** (the verifiable contract — required):
     - **Requirements**: testable statements, preferably Given/When/Then.
     - **Acceptance criteria**: a checklist where every item names its verification method (test, command, `/verify` steps, or manual check). These compile 1:1 into per-issue acceptance criteria in `/milestone-plan`.
     - **Invariants**: properties that must always hold. Flag any that could be mechanically enforced — these are the candidates for `/update-config` hooks (the environment layer).
     - **Non-goals**: explicitly out of scope.
     - **Interfaces / contracts**: APIs, data shapes, schemas (where relevant).
     - **Edge cases & error behavior**: what happens on bad input, failure, or limits.
     - **Open questions blocking speccing**: any requirement you could not make testable. If this list is non-empty, the spec is not done — surface it prominently.
   - **Proportionality**: scale the spec to the work. A medium feature may need only a half-page of acceptance criteria + invariants + non-goals; a large milestone needs the full structure. The bar is *every requirement testable*, not *maximum length*.

6. **Output destination**
   - Ask the user: output in chat or write to file?
   - Default file path if user wants a file: `brainstorm_<topic-slug>.md` in project root.
   - Topic slug: kebab-case, ASCII, max 48 chars, derived from the title.

## Constraints
- **Read-only during Phase 1**: Do not create, edit, or delete any files. Only read codebase files, CLAUDE.md, config, and web resources.
- **Project-aware**: All questions and the final design must be grounded in the actual project context (architecture, tech stack, existing patterns from CLAUDE.md and codebase).
- **One question at a time**: Never present a list of questions. Small compound questions on closely related topics are acceptable.
- **No argumentation**: Propose ideas when valuable. If the user disagrees, accept and move on.
- **Clarity score**: Must appear before every question. Score reflects how well-defined and *testable* the spec is — i.e. for how much of the work you could write a falsifiable acceptance criterion — not how many questions have been asked. The score does not have to monotonically increase — if the user adds information that introduces ambiguity or expands scope, the score can decrease accordingly. It may only reach 90 or more when every requirement is testable (see Phase 1 termination).

## Style
- Expert audience: crisp, skimmable
- Bullet lists over paragraphs in the final output
- Backticks for code elements: `file.py`, `function()`, `ClassName`
- No time estimates or emojis
- Conversational tone during Phase 1, structured document in Phase 2

## Examples
```bash
# Explicit invocation
/brainstorm Add a backoffice admin panel for managing bot content

# Invocation without args (will ask for description)
/brainstorm

# LLM auto-triggers on phrases like:
# "help me design a milestone"
# "let's brainstorm this feature"
# "I want to think through a large change"
```
