---
name: my-implement
description: |
  Implementation skill. Creates a feature branch, reads the plan from .plans/
  (fallback: plans/), and implements it task by task. Handles branch hygiene so
  my-ship works cleanly.
  Use when: "implement", "build this", "start working on the plan", "implement the plan".
allowed-tools:
  - Read
  - Write
  - Edit
  - Grep
  - Glob
  - Bash
  - AskUserQuestion
---

# Implement

Create a feature branch and implement a plan from `.plans/` at the repo root.
If `.plans/` does not exist, fall back to `plans/`.

## Conventions

- When asking questions: state the context, explain plainly, give a recommendation,
  present lettered options. One decision per question.
- Completion status: DONE | DONE_WITH_CONCERNS | BLOCKED | NEEDS_CONTEXT
- Prefer complete implementations when the marginal cost is low.
- Check if the framework has a built-in before rolling custom solutions.

---

## Step 1: Select Plan

1. List files in `.plans/` at the repo root. If none exist, check `plans/`.
2. If one plan exists, use it.
3. If multiple plans exist, present them via AskUserQuestion and ask which to implement.
4. If no plans exist: "No plans found in `.plans/` or `plans/`. Run `my-plan` first." Stop.
5. Read the selected plan fully.
6. Read the plan's `Execution Status`, `Task Checklist`, `Decisions Log`, and
   `Outcomes / Drift` sections if present. If they are missing, create them before
   implementation begins so the plan remains the execution record.

---

## Step 2: Pre-flight Checks

1. `git status` — check for uncommitted changes.
   - If dirty: AskUserQuestion — "You have uncommitted changes. A) Stash them,
     B) Commit them to current branch first, C) Abort."
2. Detect the base branch:
   - `gh repo view --json defaultBranchRef -q .defaultBranchRef.name 2>/dev/null`
   - Fallback: `main`
3. Confirm you're up to date: `git fetch origin <base> --quiet`

---

## Step 3: Create Feature Branch

1. Derive branch name from the plan filename:
   - `.plans/2026-03-30-auth-refactor.md` -> `feat/auth-refactor`
   - Strip the date prefix, kebab-case the topic, prefix with `feat/`.
2. Check the branch doesn't already exist (`git branch --list feat/<topic>`).
   - If it exists: AskUserQuestion — "Branch `feat/<topic>` already exists.
     A) Switch to it and continue, B) Use a different name, C) Abort."
3. Create and switch: `git checkout -b feat/<topic> origin/<base>`

---

## Step 4: Implement Tasks

Work through the plan's tasks in order. For each task:

1. **Read the task** — understand the files, changes, and acceptance criteria.
2. **Implement** — make the changes described. Follow existing codebase patterns.
3. **Verify acceptance criteria** — check each criterion is met.
4. **Run tests** if the task specifies them, or if a test runner is available.
5. **Commit** — one commit per task with a descriptive message:
   `<type>: <description>` (e.g., `feat: add token validation middleware`)
6. **Update the plan** — after the commit succeeds:
   - Set `Execution Status` to `In Progress` while tasks remain, then `Implemented`
     after the final task is done
   - Mark the matching checkbox in `Task Checklist` as complete
   - Record the commit SHA next to that task
   - Append any implementation-time decisions to `Decisions Log`
   - Append plan inaccuracies or controlled divergence to `Outcomes / Drift`

### Implementation rules

- Follow the plan. If you disagree with an approach, flag it via AskUserQuestion
  rather than silently diverging.
- If a task has a `[DECISION NEEDED: ...]` marker, resolve it with the user before
  implementing.
- If a task turns out to be more complex than described, break it into sub-commits
  but keep the logical grouping.
- If a test fails after implementation, fix it before moving to the next task.
- The plan is not write-once. Keep it up to date as reality changes so `my-review`,
  `my-ship`, and `my-land` inherit an accurate execution record.

---

## Step 5: Final Check

After all tasks are complete:

1. `git log origin/<base>..HEAD --oneline` — show what was committed.
2. Run the full test suite if available.
3. Report status:

```
IMPLEMENTATION COMPLETE
========================
Branch: feat/<topic>
Plan:   .plans/YYYY-MM-DD-<topic>.md (or `plans/...` compatibility path)
Tasks:  N/N completed
Commits: N

Ready for: /my-review -> /my-ship
========================
```

If any tasks were skipped or partially done, flag them clearly.

---

## Rules

- ONE commit per task. Do not batch multiple tasks into a single commit.
- Do NOT push to remote — that's `my-ship`'s job.
- Do NOT create PRs — that's `my-ship`'s job.
- If the plan references files that don't exist or paths that are wrong, investigate
  and fix rather than failing silently. Note any plan inaccuracies in the final report.
- Before finishing, ensure the plan status reads `Implemented` if all tasks are done,
  otherwise leave it at the most accurate intermediate status.
- If both `.plans/` and `plans/` exist, prefer `.plans/` unless the user explicitly
  points to a file in `plans/`.
