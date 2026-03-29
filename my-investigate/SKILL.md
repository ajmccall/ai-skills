---
name: my-investigate
description: |
  Systematic debugging with root cause investigation. Iron Law: no fixes without
  root cause. Four phases: investigate, analyze, hypothesize, implement.
  Use when: "debug this", "fix this bug", "why is this broken", "root cause".
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

# Systematic Debugging

## Iron Law

**NO FIXES WITHOUT ROOT CAUSE INVESTIGATION FIRST.**

Fixing symptoms creates whack-a-mole debugging. Every fix that doesn't address root
cause makes the next bug harder to find.

## Conventions

- When asking questions: state the context (project, branch, task), explain plainly,
  give a recommendation, present lettered options. One decision per question.
- Completion status: DONE | DONE_WITH_CONCERNS | BLOCKED | NEEDS_CONTEXT

---

## Phase 1: Root Cause Investigation

Gather context before forming any hypothesis.

1. **Collect symptoms:** Read error messages, stack traces, repro steps. If insufficient
   context, ask ONE question at a time.
2. **Read the code:** Trace the code path from symptom to potential causes. Grep for
   all references.
3. **Check recent changes:**
   ```bash
   git log --oneline -20 -- <affected-files>
   ```
   Was this working before? A regression means the root cause is in the diff.
4. **Reproduce:** Can you trigger the bug deterministically? If not, gather more evidence.

Output: **"Root cause hypothesis: ..."** — a specific, testable claim.

---

## Phase 2: Pattern Analysis

Check if the bug matches a known pattern:

| Pattern | Signature | Where to look |
|---------|-----------|---------------|
| Race condition | Intermittent, timing-dependent | Concurrent shared state |
| Nil/null propagation | NoMethodError, TypeError | Missing guards on optionals |
| State corruption | Inconsistent data, partial updates | Transactions, callbacks |
| Integration failure | Timeout, unexpected response | External API calls |
| Configuration drift | Works locally, fails elsewhere | Env vars, feature flags |
| Stale cache | Shows old data | Redis, CDN, browser cache |

Also check `git log` for prior fixes in the same area — recurring bugs in the same
files are an architectural smell.

---

## Phase 3: Hypothesis Testing

Before writing ANY fix, verify your hypothesis.

1. **Confirm:** Add a temporary log/assertion at the suspected root cause. Run repro.
   Does the evidence match?
2. **If wrong:** Return to Phase 1. Gather more evidence. Do not guess.
3. **3-strike rule:** If 3 hypotheses fail, **STOP.** AskUserQuestion:
   - A) Continue — I have a new hypothesis: [describe]
   - B) Escalate for human review
   - C) Add logging and catch it next time

**Red flags — slow down if you see:**
- "Quick fix for now" — there is no "for now." Fix right or escalate.
- Proposing a fix before tracing data flow — you're guessing.
- Each fix reveals a new problem elsewhere — wrong layer, not wrong code.

---

## Phase 4: Implementation

Once root cause is confirmed:

1. **Fix the root cause, not the symptom.** Smallest change that eliminates the problem.
2. **Minimal diff.** Fewest files, fewest lines. Resist refactoring adjacent code.
3. **Write a regression test** that fails without the fix, passes with it.
4. **Run the full test suite.** No regressions allowed.
5. **If fix touches >5 files:** AskUserQuestion about blast radius before proceeding.

---

## Phase 5: Verification & Report

**Reproduce the original bug and confirm it's fixed.** Not optional.

```
DEBUG REPORT
════════════════════════════════════════
Symptom:         [what the user observed]
Root cause:      [what was actually wrong]
Fix:             [what changed, file:line references]
Evidence:        [test output, repro showing fix works]
Regression test: [file:line of the new test]
Status:          DONE | DONE_WITH_CONCERNS | BLOCKED
════════════════════════════════════════
```

## Rules

- 3+ failed fix attempts → STOP and question the architecture.
- Never apply a fix you cannot verify.
- Never say "this should fix it." Prove it. Run the tests.
- If fix touches >5 files → ask about blast radius first.
