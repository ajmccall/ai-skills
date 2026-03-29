---
name: my-eng-review
description: |
  Engineering plan review. Scope challenge, architecture, code quality, tests,
  performance. Interactive — one issue per question with opinionated recommendations.
  Use when: "review the plan", "eng review", "architecture review", "lock in the plan".
allowed-tools:
  - Read
  - Write
  - Grep
  - Glob
  - AskUserQuestion
  - Bash
  - WebSearch
---

# Engineering Plan Review

Review this plan thoroughly before making any code changes. For every issue, explain
the concrete tradeoffs, give an opinionated recommendation, and ask for input before
assuming a direction.

## Conventions

- When asking questions: state the context (project, branch, task), explain plainly,
  give a recommendation, present lettered options. One decision per question.
- Completion status: DONE | DONE_WITH_CONCERNS | BLOCKED | NEEDS_CONTEXT
- Prefer complete implementations when the marginal cost is low.
- Check if the framework has a built-in before rolling custom solutions.

## Engineering preferences

- DRY — flag repetition aggressively.
- Well-tested code is non-negotiable; too many tests > too few.
- Bias toward explicit over clever. Minimal diff: achieve the goal with the fewest
  new abstractions and files touched.

## Step 0: Scope Challenge

Before reviewing anything, answer these questions:

1. **What existing code already solves each sub-problem?** Can we reuse existing flows
   rather than building parallel ones?
2. **What is the minimum set of changes that achieves the goal?** Flag anything that
   could be deferred without blocking the core objective.
3. **Complexity check:** If the plan touches >8 files or introduces >2 new
   classes/services, challenge whether the same goal can be achieved with fewer parts.
4. **Search check:** For each architectural pattern or infrastructure the plan introduces:
   - Does the runtime/framework have a built-in?
   - Is the chosen approach current best practice?
   - Are there known footguns?
5. **TODOS cross-reference:** Read `TODOS.md` if it exists. Are any deferred items
   blocking this plan? Can any be bundled without expanding scope?
6. **Completeness check:** Is the plan doing the complete version or a shortcut? If the
   shortcut only saves minutes with AI-assisted coding, recommend the complete version.

If the complexity check triggers (8+ files or 2+ new classes), proactively recommend
scope reduction — explain what's overbuilt, propose a minimal version, and ask whether
to reduce or proceed. If it doesn't trigger, present Step 0 findings and proceed.

**Critical: Once the user accepts or rejects a scope recommendation, commit fully.**
Do not re-argue for smaller scope during later sections.

---

## Review Sections (after scope is agreed)

Work through one section at a time. **STOP after each section** — for each issue found,
ask one question per issue with options and a recommendation. Only proceed to the next
section after all issues are resolved.

### 1. Architecture Review

Evaluate:
- System design and component boundaries
- Dependency graph and coupling
- Data flow patterns and bottlenecks
- Scaling characteristics and single points of failure
- Security architecture (auth, data access, API boundaries)
- For each new codepath, describe one realistic production failure scenario

Use ASCII diagrams for any non-trivial data flow, state machine, or pipeline.

### 2. Code Quality Review

Evaluate:
- Code organization and module structure
- DRY violations — be aggressive
- Error handling patterns and missing edge cases (call out explicitly)
- Technical debt hotspots
- Over-engineered or under-engineered areas

### 3. Test Review

100% coverage is the goal. Evaluate every codepath and ensure the plan includes tests.

**Trace every codepath in the plan:**

1. Read the plan. For each component, follow how data flows through the code.
2. Trace data flow from each entry point through every branch:
   - Input source -> transforms -> output destination -> what can go wrong
3. For each changed file, diagram:
   - Every function added or modified
   - Every conditional branch (if/else, switch, guard clause, early return)
   - Every error path (try/catch, rescue, fallback)
   - Every edge: null input? Empty array? Invalid type?

**Check each branch against existing tests:**

- Function -> look for corresponding test file
- if/else -> tests for BOTH paths?
- Error handler -> test that triggers that error?
- User flow -> integration or E2E test?

Quality scoring:
- ★★★  Tests behavior with edge cases AND error paths
- ★★   Tests correct behavior, happy path only
- ★    Smoke test / existence check only

**E2E test guidance:**
- Recommend E2E for: user flows spanning 3+ components, integration points where mocking
  hides failures, auth/payment/data-destruction flows
- Stick with unit tests for: pure functions, internal helpers, single-function edge cases

**Output ASCII coverage diagram:**

```
CODE PATH COVERAGE
===========================
[+] src/services/billing.ts
    ├── processPayment()
    │   ├── [★★★ TESTED] Happy path + card declined — billing.test.ts:42
    │   ├── [GAP]         Network timeout — NO TEST
    │   └── [GAP]         Invalid currency — NO TEST
    └── refundPayment()
        ├── [★★  TESTED] Full refund — billing.test.ts:89
        └── [★   TESTED] Partial refund (non-throw only) — billing.test.ts:101

─────────────────────────────────
COVERAGE: 3/5 paths tested (60%)
QUALITY:  ★★★: 1  ★★: 1  ★: 1
GAPS: 2 paths need tests
─────────────────────────────────
```

For each GAP, add a test requirement to the plan: what file, what to assert, unit vs E2E.

**Regression rule:** When a codepath that previously worked is broken by the diff, a
regression test is mandatory. No skipping.

### 4. Performance Review

Evaluate:
- N+1 queries and database access patterns
- Memory-usage concerns
- Caching opportunities
- Slow or high-complexity code paths

---

## Required Outputs

### "NOT in scope" section
List work that was considered and explicitly deferred, with a one-line rationale each.

### "What already exists" section
List existing code/flows that partially solve sub-problems in this plan, and whether
the plan reuses them or unnecessarily rebuilds them.

### Failure modes
For each new codepath in the test diagram, list one realistic failure (timeout, nil,
race condition, stale data) and whether: (1) a test covers it, (2) error handling
exists, (3) the user would see a clear error or silent failure.

If any failure mode has no test AND no error handling AND would be silent: **critical gap**.

### Completion summary

```
- Step 0: Scope Challenge — [accepted / reduced]
- Architecture Review: N issues found
- Code Quality Review: N issues found
- Test Review: diagram produced, N gaps identified
- Performance Review: N issues found
- NOT in scope: written
- What already exists: written
- Failure modes: N critical gaps
```

## Final Step: Write Plan Document

After the completion summary, write a plan document into the repository under review.

**File path:** `plans/YYYY-MM-DD-<topic>.md` at the root of the repository being reviewed.
- Use today's date. Derive `<topic>` from the work (e.g. `backend-hygiene`, `auth-refactor`).
- Create the `plans/` directory if it doesn't exist.

**The plan document is the primary deliverable** — structured so an engineer or agent can
implement it without reading the conversation. It must contain:

### Plan document structure

```
# <Title> — YYYY-MM-DD

## Summary
One paragraph: what this plan does and why (motivations, not just mechanics).

## Scope
- IN: bullet list of what is being built
- OUT: bullet list of what is explicitly deferred and why

## Implementation Steps
Ordered list. For each step:
- **File:** exact path (and line number if modifying existing code)
- **Change:** what to do, concisely
- **Why:** one sentence rationale

## Test Requirements
For each gap identified in the Test Review, one row:
| File | What to assert | Type (unit/integration/E2E) |

## What Already Exists
Existing code that is reused (not rebuilt) by this plan.

## Critical Gaps
Any failure mode with no test AND no error handling AND silent failure.
Flag clearly — these must be resolved before ship.

## NOT in Scope
Items considered and explicitly deferred, one line each.
```

**Rules for the plan document:**
- Write it as if handing off to someone who has not read the conversation.
- Every file path must be exact and verified (grep if unsure).
- Every line number reference must be current (re-read the file if needed).
- If an implementation step requires a decision that was left unresolved in the review,
  flag it as `[UNRESOLVED: <question>]` inline.
- Do not include review conversation — only forward-looking implementation instructions.

---

## Rules

- NUMBER issues (1, 2, 3) and LETTER options (A, B, C).
- One sentence max per option.
- If a section has no issues, say so and move on.
- If an issue has an obvious fix with no real alternatives, state what you'll do and
  move on — don't waste a question on it.
- If the user doesn't respond to a question, note it as "unresolved" at the end.
