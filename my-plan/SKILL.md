---
name: my-plan
description: |
  Implementation planning. Reads a spec from specs/ and eng review from reviews/,
  produces a prescriptive implementation plan in .plans/ (fallback: plans/).
  Output is concrete enough
  to implement without further context: ordered tasks, exact files, acceptance criteria.
  Use when: "plan this", "make the plan", "implementation plan", "break this down".
allowed-tools:
  - Read
  - Write
  - Grep
  - Glob
  - Bash
  - AskUserQuestion
---

# Implementation Planning

Produce a prescriptive implementation plan from a spec and its eng review. The plan
must be concrete enough that an engineer (or agent) can implement it step by step
without reading any prior conversation.

**This skill does NOT review or ideate.** It synthesizes decisions already made into
an ordered, file-level implementation plan.

## Conventions

- When asking questions: state the context, explain plainly, give a recommendation,
  present lettered options. One decision per question.
- Completion status: DONE | DONE_WITH_CONCERNS | BLOCKED | NEEDS_CONTEXT

---

## Step 1: Gather Inputs

1. Look in `specs/` for the relevant spec file. If multiple exist, pick the most
   recent or the one matching the user's request.
2. Look in `reviews/` for a corresponding eng review. If none exists, warn:
   "No eng review found — proceeding from spec only. Run `my-eng-review` first
   for better results."
3. Read `CLAUDE.md`, `TODOS.md` if they exist.
4. `git log --oneline -20` for recent context.

If no spec is found, AskUserQuestion: "I don't see a spec in `specs/`. Point me to
one, or should we run `my-ideate` first?"

---

## Step 2: Resolve Unresolved Questions

Scan the spec and review for any `[UNRESOLVED: ...]` markers.

For each:
1. Investigate the codebase (grep, read files) to see if the answer is obvious.
2. If it is, state the resolution and move on.
3. If not, AskUserQuestion with options and a recommendation.

Do NOT proceed to planning with unresolved questions — every decision must be locked
in before writing the plan.

---

## Step 3: Map the Codebase

For each component mentioned in the spec:

1. Find the exact files that will be created or modified (Glob + Grep).
2. Read each file to understand current structure.
3. Identify:
   - Existing patterns to follow (how similar things are done in this codebase)
   - Shared code to reuse
   - Files that will need coordinated changes (e.g., a type change that propagates)

---

## Step 4: Build Task Breakdown

Break the work into ordered implementation tasks. Each task should be:
- **Independently testable** — you can verify it works before moving to the next
- **Small enough to review** — ideally one logical change
- **Ordered by dependency** — later tasks can depend on earlier ones

For each task:
- What files to create/modify (exact paths)
- What specifically to change (function signatures, new modules, config changes)
- Acceptance criteria (what "done" looks like for this task)
- Which review concerns this task addresses (if any)

Group related changes into the same task when they must ship together (e.g., a
migration and the code that depends on it).

---

## Step 5: Test Strategy

For each task, specify what tests are needed:

1. Pull test requirements from the eng review if available.
2. For each new codepath, specify:
   - Test file path (follow existing test conventions in the repo)
   - What to assert
   - Type: unit / integration / E2E
3. If the repo has a test runner, note the exact command to run.

---

## Step 6: Write Plan Document

**File path:** `.plans/YYYY-MM-DD-<topic>.md` at the repo root.
- Use today's date. Derive `<topic>` from the spec.
- Create `.plans/` if it doesn't exist.

### Plan document structure

```markdown
# <Title> — Implementation Plan — YYYY-MM-DD

## Coordination
- Root repo owner: <agent or name>
- Backend branch: `<branch>` or `N/A`
- Mobile branch: `<branch>` or `N/A`
- Worktree roots: `<path>` or `N/A`
- Related repos: backend, mobile, or `N/A`

## Source
- Spec: `specs/YYYY-MM-DD-<topic>.md`
- Review: `reviews/YYYY-MM-DD-<topic>.md` (or "none")

## Summary
One paragraph: what is being built, why, and the chosen approach.

## Scope
- **IN:** what is being built
- **OUT:** what is explicitly deferred (from spec + review)

## Tasks

### Task 1: <name>
**Files:**
- `path/to/file.ts` — create / modify (line ~N)
- `path/to/other.ts` — modify (line ~N)

**Changes:**
- Concrete description of what to do
- Include function names, type signatures, key logic

**Tests:**
- `path/to/file.test.ts` — assert X (unit)

**Acceptance criteria:**
- [ ] Criterion 1
- [ ] Criterion 2

### Task 2: <name>
**Depends on:** Task 1
...

## Critical Gaps
Carried forward from review. Each must be addressed by a specific task above.
If any are not addressed, flag as `[UNADDRESSED: <gap>]`.

## NOT in Scope
Items deferred, one line each.

## Execution Status
Status: Planned

## Task Checklist
- [ ] Task 1: <name>
  - Commit:
- [ ] Task 2: <name>
  - Commit:

## Decisions Log
- YYYY-MM-DD: Plan created from spec and review.

## Outcomes / Drift
- None yet.
```

**Rules for the plan document:**
- Write as if handing off to someone who has not read any conversation.
- Every file path must be exact and verified (grep/glob if unsure).
- Every line number reference must be current (re-read the file).
- No vague steps. "Refactor the auth layer" is not a task. "Add `validateToken()`
  to `src/auth/middleware.ts` that checks JWT expiry and returns 401" is a task.
- If a task requires a judgment call you can't make, flag as `[DECISION NEEDED: ...]`.
- The `Execution Status`, `Task Checklist`, `Decisions Log`, and `Outcomes / Drift`
  sections are living workflow sections. Create them now so downstream skills can
  update them without rewriting the plan.
- Include a `## Coordination` block near the top so cross-repo work can declare
  branch names, worktree roots, and ownership in the same file.
- In `Task Checklist`, include one checkbox per numbered task using the same task
  names as the main `Tasks` section so downstream skills can mark progress cleanly.
- Do not include review commentary outside `Critical Gaps` and the living workflow
  sections — the main body should remain forward-looking implementation instructions.

Present the plan to the user via AskUserQuestion:
- A) Approve — ready to implement
- B) Revise — specify which tasks need changes
- C) Reconsider — go back to a specific step

If the repo already documents `.plans/` as the implementation-plan location, treat that
as canonical.

---

## Rules

- NUMBER tasks (1, 2, 3) and LETTER options (A, B, C).
- STOP after Step 2 (resolving questions) before proceeding — confirm all decisions.
- If a task is trivial (< 5 lines, obvious change), still list it — completeness
  matters more than brevity in a plan.
- If the spec is too vague to produce concrete tasks, say so and recommend running
  `my-eng-review` or revising the spec rather than guessing.
