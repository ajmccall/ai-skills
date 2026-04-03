---
name: my-review
description: |
  Pre-landing diff review. Analyzes branch diff for structural issues that
  tests don't catch: SQL safety, race conditions, scope drift, dead code,
  test gaps. Auto-fixes what it can, asks about the rest.
  Use when: "review this", "code review", "pre-landing review", "check my diff".
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Grep
  - Glob
  - AskUserQuestion
  - WebSearch
---

# Pre-Landing Diff Review

Analyze the current branch's diff against the base branch for structural issues
that tests don't catch.

## Conventions

- When asking questions: state the context (project, branch, task), explain plainly,
  give a recommendation, present lettered options. One decision per question.
- Completion status: DONE | DONE_WITH_CONCERNS | BLOCKED | NEEDS_CONTEXT

---

## Step 0: Detect base branch

1. `gh pr view --json baseRefName -q .baseRefName`
2. Fallback: `gh repo view --json defaultBranchRef -q .defaultBranchRef.name`
3. Fallback: `main`

---

## Step 1: Check branch

1. `git branch --show-current`
2. If on base branch or no diff against base: "Nothing to review." Stop.
3. `git fetch origin <base> --quiet && git diff origin/<base> --stat`

---

## Step 1.5: Scope Drift Detection

Before reviewing code quality, check: did they build what was requested?

1. Read `TODOS.md` (if exists), PR description (`gh pr view --json body -q .body`),
   commit messages (`git log origin/<base>..HEAD --oneline`).
2. Check for a plan file: search `.plans/*.md` at the repo root first, then
   `plans/*.md` as a compatibility fallback, for files matching the branch name or
   topic. If found, extract actionable tasks and cross-reference against the diff.
3. If a matching plan is found, read its `## Coordination` block and verify the
   branch/worktree ownership matches the repo you are reviewing.
4. Identify **stated intent** vs **actual diff**.
5. Flag:
   - **Scope creep:** Files changed unrelated to stated intent, features not mentioned
   - **Missing requirements:** Stated work not addressed in the diff
6. If a plan file is found, treat it as a living execution record:
   - Read `Execution Status`, `Task Checklist`, `Decisions Log`, and `Outcomes / Drift`
     - Cross-check them against the actual diff before trusting them
7. Output:
   ```
   Scope Check: CLEAN / DRIFT DETECTED / REQUIREMENTS MISSING
   Intent: <1-line summary>
   Delivered: <1-line summary>
   ```
   This is informational — does not block the review.

---

## Step 2: Get the diff

```bash
git fetch origin <base> --quiet
git diff origin/<base>
```

Read the full diff. For large diffs, also review full files (not just hunks) for context.

---

## Step 3: Two-pass review

### Pass 1 — CRITICAL (these can cause incidents)

**SQL & Data Safety:**
- Raw SQL with string interpolation → SQL injection risk
- Destructive migrations without reversibility
- Missing transaction boundaries on multi-step writes
- N+1 queries introduced by new associations

**Race Conditions & Concurrency:**
- Shared mutable state without locking
- Check-then-act patterns (TOCTOU)
- Missing uniqueness constraints at DB level (relying only on app validation)

**Security Boundaries:**
- User input flowing to system commands, HTML, or SQL without sanitization
- Missing authorization checks on new endpoints
- Secrets or credentials in code

**Enum & Value Completeness:**
When the diff introduces a new enum value, status, or type: use Grep to find ALL
references to sibling values. Check if the new value is handled everywhere. This
requires reading code OUTSIDE the diff.

### Pass 2 — INFORMATIONAL (code quality, not incidents)

- Conditional side effects (hidden mutations in unexpected places)
- Magic numbers and string coupling
- Dead code and stale comments
- Test gaps (see Step 4)
- Performance concerns (large allocations, unbounded queries)

---

## Step 4: Test Coverage Diagram

Trace every codepath changed in the diff. For each changed file:

1. Read the full file, trace data flow through every branch
2. Check each branch against existing tests
3. Rate: ★★★ (edges+errors), ★★ (happy path), ★ (smoke)

Output ASCII diagram:
```
CODE PATH COVERAGE
===========================
[+] src/services/billing.ts
    ├── processPayment()
    │   ├── [★★★ TESTED] Happy path + declined — billing.test.ts:42
    │   └── [GAP]         Timeout — NO TEST
─────────────────────────────────
COVERAGE: N/M paths tested
GAPS: K paths need tests
─────────────────────────────────
```

**Regression rule:** If the diff breaks existing behavior, flag it as critical.

---

## Step 5: Fix-First Flow

For each finding:

1. **AUTO-FIX** (mechanical, no judgment needed): Fix it, commit as
   `fix: <description>`. Examples: dead code removal, missing null check,
   obvious N+1.
2. **ASK** (requires judgment): AskUserQuestion with options. Examples:
   architecture decisions, trade-off choices, scope questions.

After all fixes: re-run `git diff origin/<base> --stat` to confirm changes.

## Step 5.5: Update Plan Record

If a matching plan file was found, update it before finishing:

1. Set `Execution Status` to:
   - `Reviewed` if no issues remain
   - `Review Changes Applied` if you auto-fixed findings
   - leave a more conservative status if review is blocked
2. Update the `## Coordination` block if branch ownership, worktree roots, or repo
   split changed during the review.
3. In `Task Checklist`, note any task that is still incomplete or contradicted by the diff.
4. Append material review outcomes to `Decisions Log` when a fix changed behavior.
5. Append findings, scope drift, and fix commit SHAs to `Outcomes / Drift`.

The goal is that the plan remains the canonical record of what was intended, what
actually shipped on the branch, and what still needs attention.

---

## Step 6: Output

```
REVIEW SUMMARY
═══════════════════════════════════
Branch:     <branch> → <base>
Files:      <N files changed>
Scope:      <CLEAN / DRIFT / MISSING>

CRITICAL findings:    N (M auto-fixed)
INFORMATIONAL:        N (M auto-fixed)
Test coverage:        N/M paths (X%)

Findings:
  [CRITICAL] [FIXED] SQL injection in user_controller.rb:45
  [CRITICAL] [ASK]   Race condition in payment_service.rb:112
  [INFO]     [FIXED] Dead code removed from helpers.rb
  [INFO]     [GAP]   Missing test for error path in auth.rb:67

VERDICT: CLEAR / ISSUES_FOUND
═══════════════════════════════════
```
