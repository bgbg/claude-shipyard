---
description: Create a Marp slide deck from scratch. Gathers topic info through guided Q&A, writes an outline for review, then generates a full presentation in strict Marp format.
---

# Marp Presentation Builder

Guide the user from a raw topic idea to a finished Marp slide deck. Work in three phases: clarification → outline → deck.

## Phase 1: Clarification (read-only)

### Step 1 — Receive initial description

Accept the lesson topic from `{{ARGS}}` or from conversation context. If nothing was provided, ask:

> "What is the topic of your presentation?"

### Step 2 — Anchor with "why"

The first question must establish the motivation and goals behind the presentation.
Display clarity score before the question: `**Clarity: N/100**`

### Step 3 — Iterative clarification loop

- Ask ONE question at a time (small, closely related compound questions are acceptable — never disguise multiple large questions as one).
- Before each question, display: `**Clarity: N/100**`
- Start with "why" (goals, motivation), then target the largest remaining gap in understanding. Dimensions to cover: audience, learning objectives, scope, constraints, subtopics, tone, code/examples, visuals, format.
- Be proactive: read relevant files, search the web — do whatever is needed to ask informed questions and propose concrete ideas.
- Propose ideas and directions for the user to react to when it adds value. Act as a collaborator, not an interviewer.
- **Do NOT modify any files during this phase.**

### Termination conditions

- **Score reaches 99**: Say "I have all the information I need. Is there anything else you want to add?" Then proceed to Phase 2 after user confirms.
- **User says stop / enough / go ahead**: Proceed to Phase 2 immediately.
- **10 questions asked**:  Say "I've asked enough questions. Proceeding to Phase 2." Then proceed regardless of score.



## Phase 2: Outline

### Build the outline

Using all gathered information, write a structured outline as a Markdown file. Save it to `<slugified-topic>-outline.md` in the current directory.

Outline format:

```markdown
# <Presentation Title>

**Audience:** …
**Objective:** …
**Length:** ~N slides

---

## Agenda
1. Section One Title
2. Section Two Title
3. Section Three Title
…

---

## 1. Section One Title
- Subtopic A
  - Detail / example idea
- Subtopic B
- Subtopic C

## 2. Section Two Title
- Subtopic A
- Subtopic B

…

## Summary / Key Takeaways
- Key point 1
- Key point 2
- Key point 3
```

After saving, tell the user:

> "I've written the outline to `<filename>`. Please review it, edit freely, then reply **'approved'** (or just **'ok'**) when you're ready for the full deck."

**Do NOT proceed to Phase 3 until the user explicitly approves.**

## Phase 3: Marp Deck

### Slide structure rules

Follow this exact structure — every rule is mandatory:

1. **Front matter** — always first:
   ```
   ---
   marp: true
   theme: default
   paginate: true
   ---
   ```

2. **Title slide** — no page number:
   ```markdown
   <!-- _paginate: false -->

   # Presentation Title

   Subtitle or author line
   ```

3. **Agenda slide** — immediately after title:
   ```markdown
   ---

   # Agenda

   1. Section One
   2. Section Two
   3. Section Three
   ```

4. **Section intro slide** — before every section, show the full agenda and **bold** the upcoming section:
   ```markdown
   ---

   # Agenda

   1. Section One
   2. **Section Two** ← next
   3. Section Three
   ```

5. **Section heading slide** — use `#` for each major section:
   ```markdown
   ---

   # Section Title
   ```

6. **Subtopic slides** — use `##` for each subtopic:
   ```markdown
   ---

   ## Subtopic Title

   - Bullet one
   - Bullet two
   - Bullet three
   ```

7. **Example / code / diagram slides** — use `###`:
   ```markdown
   ---

   ### Example: descriptive title

   ```language
   // code here — max ~6 lines
   ```
   ```

   **Code example rules:**
   - Where coding examples are relevant, use them extensively — a concrete example beats a bullet point.
   - Every code block must fit on one screen. If a realistic snippet is too long, trim it: use `# ...` or `// ...` ellipsis comments, placeholder function calls, or dummy values to represent the omitted parts. Incomplete but readable beats complete but unreadable.
   - Code does not need to be fully runnable. Pseudocode, partial snippets, and illustrative stubs are fine as long as the key concept is clear.
   - Split long examples across sequential `###` slides rather than shrinking font or overflowing.

   For diagrams, always use Mermaid fenced code blocks:
   ```markdown
   ---

   ### Diagram: descriptive title

   ```mermaid
   graph TD
     A[Start] --> B[Step]
     B --> C[End]
   ```
   ```
   Use the most appropriate Mermaid diagram type for the concept: `graph`/`flowchart` for flows and architecture, `sequenceDiagram` for interactions, `classDiagram` for structure, `gitGraph` for branching, `timeline` for chronology.

   **Diagram sizing rule:** Every diagram must fit on one screen — never let it overflow. If a concept is too complex for a single slide, use a **progressive reveal pattern**: show a high-level diagram with collapsed "meta" nodes first, then dedicate follow-up slides to zoomed-in sub-diagrams that expand each meta node. This keeps every slide readable while still covering the full depth.

   **Screenshot placeholders:** When a screenshot would help (UI walkthrough, tool output, browser result, etc.), leave an explicit placeholder instead of skipping it. Use this format:
   ```markdown
   ---

   ### Screenshot: descriptive title

   ![screenshot: what to capture here](screenshot-placeholder.png)

   > **TODO:** Replace with screenshot of [specific thing to capture]
   ```
   Be specific in the TODO note — describe exactly what should be in the screenshot so the presenter knows what to capture.

8. **Slide density** — keep slides compact:
   - Text slides: 3–6 bullets
   - Code slides: ~6 lines of code
   - Never cram more than one concept per slide

9. **Summary / Key Takeaways** — always the final slide:
   ```markdown
   ---

   # Summary / Key Takeaways

   - Takeaway 1
   - Takeaway 2
   - Takeaway 3
   ```

### Generate and save

Write the complete Marp deck to `<slugified-topic>.md` in the current directory.

### Serve live

After saving, start the Marp live server in the background:

```bash
marp --server <slugified-topic>.md
```

This launches a browser preview at `http://localhost:8080` that auto-refreshes on every save. Tell the user:

> "Your deck is live at http://localhost:8080 — edit `<filename>.md` and the browser will reload automatically. Press Ctrl+C in the terminal to stop the server."
