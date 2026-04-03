---
name: my-design-review
description: |
  Design and UX review of a spec before implementation. Reviews product flows,
  information architecture, interaction states, accessibility expectations, and
  visual specificity, then records findings and unresolved questions in reviews/.
  Use when: "design review", "review the design", "review the spec design",
  "check the UX plan".
allowed-tools:
  - Read
  - Write
  - Edit
  - Grep
  - Glob
  - Bash
  - AskUserQuestion
  - WebSearch
---

# Design Review

Review a spec from a product-design perspective before implementation.

**This skill is review-only.** It does not update `specs/` or `.plans/`. Its job is
to identify design gaps, clarify what is obvious, and record design findings plus
open questions in `reviews/` so ideation, product, and engineering can refine the
source artifacts intentionally.

## Conventions

- When asking questions: state context, explain plainly, give a recommendation,
  present lettered options. One decision per question.
- Completion status: DONE | DONE_WITH_CONCERNS | BLOCKED | NEEDS_CONTEXT
- The design review output belongs in `reviews/`, not in the spec or plan.

---

## Step 0: Gather Inputs

1. Read `AGENTS.md` and `DESIGN.md` if they exist.
2. Find the relevant spec in `specs/`.
3. Find the matching review in `reviews/` if it exists.
4. Read `git log --oneline -10` for recent context.

If no spec exists, stop and ask the user whether to point you at one or run `my-ideate`
first.

Treat the spec as product intent and the existing review, if present, as prior review
context. Do not rewrite either one directly from this skill.

**UI scope check:** If the spec has no meaningful UI or UX surface, say so and exit:
"No UI scope — design review not applicable."

---

## Step 1: Review Design Intent

Review the spec and identify what a user would experience, not just what the system
would do.

Check:
- what the user sees first, second, and third on each key screen or flow
- whether the primary path is obvious
- whether important secondary actions are discoverable but not competing
- whether the spec defines interaction patterns concretely enough to implement

If the structure is vague, write a design finding in the review and, when useful,
include a short ASCII outline of the intended hierarchy.

**STOP.** AskUserQuestion for material design decisions that are not obvious.

---

## Step 2: Review States and Edge UX

Check whether the spec defines:
- loading, empty, error, success, and partial states
- validation and recovery paths
- long or unusual content lengths
- first-run and no-data experiences
- accessibility expectations that materially affect the flow

If the spec leaves these undefined:
- add a design finding to the review
- add an unresolved question when a product or design decision is needed

Do not invent product behavior silently.

**STOP.** AskUserQuestion for material gaps that block implementation clarity.

---

## Step 3: Review Visual Specificity

Check whether the spec is concrete enough to avoid generic output.

Look for:
- vague phrases like "clean modern UI" without details
- generic card-grid defaults
- unclear hierarchy or layout intent
- unspecified typography, spacing, density, or tone
- responsive behavior described too loosely

Flag "AI slop" patterns when the spec is drifting toward generic generated design.
Turn vague taste language into review findings and unresolved questions, not silent
spec edits.

**STOP.** AskUserQuestion for taste or direction decisions that need explicit choice.

---

## Step 4: Review Design-to-Engineering Implications

Call out the implementation consequences of the design:
- state handling the UI will require
- API or payload assumptions implied by the UX
- accessibility requirements that affect implementation
- responsive behavior that must be supported explicitly
- test cases implied by the user flow

These belong in the review artifact so `my-plan` can turn them into tasks later.

If the issue is unresolved, mark it as:
- `[UNRESOLVED: <question>]`

If the answer is obvious from the spec, record the recommendation clearly in the
design review section.

---

## Step 5: Write Review Output

Write the design review into the matching file in `reviews/`.

If a matching engineering review already exists:
- append or update a dedicated `## Design Review` section in that file

If no review file exists yet:
- create `reviews/YYYY-MM-DD-<topic>.md`
- include enough structure that engineering review and design review can coexist in
  the same file later

Use this structure inside the review artifact:

```markdown
## Design Review

### Strengths
- What is already clear and implementation-ready from a UX/design perspective

### Design Gaps
- Concrete UX, IA, state, accessibility, or specificity gaps

### Design-to-Engineering Implications
- UI-driven constraints that planning and implementation must account for

### Unresolved Questions
- [UNRESOLVED: ...]
```

The output should be specific enough that `my-plan` can consume it without reading
the conversation.

---

## Final Output

Report the result in this format:

```text
DESIGN REVIEW SUMMARY
========================
Spec:      specs/YYYY-MM-DD-<topic>.md
Review:    reviews/YYYY-MM-DD-<topic>.md
Findings:  N
Open questions: N

Result:
  [CLEAR]      <summary>
  [GAP]        <summary>
  [UNRESOLVED] <summary>

Ready for: ideation/spec refinement | my-plan
========================
```

---

## Rules

- Do not update `specs/` from this skill.
- Do not update `.plans/` from this skill.
- Do not use numeric 1-10 scorecards.
- Persist design findings in `reviews/`.
- Use `[UNRESOLVED: ...]` markers for design decisions that still need a choice.
- Keep the "stop and ask" flow for material unresolved design questions.
