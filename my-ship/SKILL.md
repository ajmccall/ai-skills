---
name: my-ship
description: |
  Ship workflow: merge base branch, run tests, audit coverage, bump version,
  update changelog, commit, push, create PR. Non-interactive — runs straight
  through, only stops for real blockers.
  Use when: "ship", "create a PR", "push to main", "ship it".
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Grep
  - Glob
  - Agent
  - AskUserQuestion
---

# Ship: Automated Ship Workflow

Non-interactive. Do NOT ask for confirmation at any step. The user said `/my-ship`
which means DO IT. Run straight through and output the PR URL at the end.

## Conventions

- When asking questions: state the context (project, branch, task), explain plainly,
  give a recommendation, present lettered options. One decision per question.
- Completion status: DONE | DONE_WITH_CONCERNS | BLOCKED | NEEDS_CONTEXT
- Prefer complete implementations when the marginal cost is low.

**Only stop for:**
- On the base branch (abort)
- Merge conflicts that can't be auto-resolved
- In-branch test failures
- MINOR or MAJOR version bump needed (ask)

**Never stop for:**
- Uncommitted changes (always include them)
- Version bump choice (auto-pick PATCH)
- CHANGELOG content (auto-generate from diff)
- Commit message approval (auto-commit)

---

## Step 0: Detect base branch

1. `gh pr view --json baseRefName -q .baseRefName` — if PR exists, use that branch.
2. If no PR: `gh repo view --json defaultBranchRef -q .defaultBranchRef.name`
3. Fallback: `main`

Use "the base branch" in all subsequent commands.

---

## Step 1: Pre-flight

1. Check current branch. If on the base branch, **abort**: "Ship from a feature branch."
2. `git status` (never `-uall`). Uncommitted changes are always included.
3. `git diff <base>...HEAD --stat` and `git log <base>..HEAD --oneline` to understand
   what's being shipped.
4. Find the active plan in `.plans/` first, then `plans/` as a compatibility fallback.
   Read its `Execution Status`, `Task Checklist`, `Decisions Log`, and `Outcomes / Drift`
   so the PR and shipping report reflect the real plan state.

---

## Step 2: Merge base branch

```bash
git fetch origin <base> && git merge origin/<base> --no-edit
```

If merge conflicts: try auto-resolve for simple cases (VERSION, CHANGELOG ordering).
If complex, **STOP** and show them.

If already up to date: continue silently.

---

## Step 3: Run tests

Detect the project's test command:
1. Read CLAUDE.md for a `## Testing` section
2. Auto-detect: look for `package.json` scripts (test), `Makefile` targets, `Gemfile`
   (rails test / rspec), `pytest.ini`, `go.mod` (go test), `Cargo.toml` (cargo test)
3. If no test infrastructure found, skip with note: "No test framework detected."

Run the detected test command.

### Test Failure Triage

If tests fail, classify each failure:

**In-branch** if: the failing test or the code it tests was modified on this branch,
OR the failure traces to a change in the branch diff.

**Pre-existing** if: neither the test nor the code it tests was modified, AND the
failure is unrelated to any branch change. When ambiguous, default to in-branch.

**In-branch failures → STOP.** Show them. Developer must fix before shipping.

**Pre-existing failures → AskUserQuestion:**
- A) Investigate and fix now
- B) Add as TODO and ship anyway
- C) Skip — ship anyway

If all failures are pre-existing and handled, continue. If any in-branch failures
remain, do not proceed.

**If all pass:** Note counts briefly, continue.

---

## Step 3.4: Test Coverage Audit

Evaluate what was ACTUALLY coded (from the diff), not what was planned.

**1. Trace every codepath changed** using `git diff origin/<base>...HEAD`:

Read every changed file. For each, trace data flow through every branch:
- Entry points -> conditional branches -> error paths -> edge cases
- For each branch: is there a test? Rate: ★★★ (edges+errors), ★★ (happy path), ★ (smoke)

**2. Output ASCII coverage diagram:**

```
CODE PATH COVERAGE
===========================
[+] src/services/billing.ts
    ├── processPayment()
    │   ├── [★★★ TESTED] Happy path + card declined — billing.test.ts:42
    │   └── [GAP]         Network timeout — NO TEST
    └── refundPayment()
        └── [★★  TESTED] Full refund — billing.test.ts:89

─────────────────────────────────
COVERAGE: 2/3 paths tested (67%)
GAPS: 1 path needs tests
─────────────────────────────────
```

**3. Generate tests for uncovered paths** (if test framework exists):
- Prioritize error handlers and edge cases first
- Read 2-3 existing test files to match conventions
- Write tests, run each. Passes → commit as `test: coverage for {feature}`. Fails → fix once, still fails → revert.
- Cap: 20 tests generated max, 2-min per-test exploration cap.

**Regression rule:** If the diff breaks existing behavior, write a regression test
immediately. No skipping.

**If diff is test-only:** Skip this step.

---

## Step 4: Version bump

If `VERSION` file exists:
- Auto-pick PATCH bump for most changes
- If the diff introduces a breaking change or major new feature, ask about MINOR/MAJOR
- Update the VERSION file

If no VERSION file: skip.

---

## Step 5: CHANGELOG

If `CHANGELOG.md` exists:
- Read `git log <base>..HEAD --oneline` to understand all changes
- Write a new entry: lead with what the user can now DO, not implementation details
- Use plain language: "You can now..." not "Refactored the..."

If no CHANGELOG: skip.

---

## Step 6: Stage and commit

1. Stage all changes. Prefer specific filenames over `git add -A`.
2. If multiple logical changes exist, split into bisectable commits:
   - Separate: renames from behavior changes, tests from features, docs from code
3. Commit messages: concise, conventional format (`feat:`, `fix:`, `test:`, `chore:`)

---

## Step 7: Push

```bash
git push origin HEAD -u
```

If push fails due to remote changes: `git pull --rebase origin <current-branch>` then retry.

---

## Step 8: Create PR

```bash
gh pr create --title "<title>" --body "$(cat <<'EOF'
## Summary
<1-3 bullet points of what changed and why>

## Test coverage
<coverage summary from Step 3.4: N paths tested, M gaps, K tests generated>

## Changes
<list of key files changed with brief descriptions>

EOF
)"
```

- PR title: short (<70 chars), describes the change
- The PR body should explicitly note that the PR is intended to land via **squash merge**
- If a plan exists, include its path in the PR body and summarize the final checklist state
- If a PR already exists for this branch, skip creation and just push

## Step 8.5: Update Plan Record

If a matching plan exists, update it before finishing:

1. Set `Execution Status` to `Shipped`
2. Update `Task Checklist` to reflect the final implemented state
3. Append PR URL, test summary, and coverage summary to `Outcomes / Drift`
4. Append any release-time decisions (version bump, deferred gaps) to `Decisions Log`

---

## Output

Print the PR URL. Done.

```
STATUS: DONE
PR: <url>
Branch: <feature> → <base>
Tests: <pass count> | Coverage audit: <N paths, M tested>
Version: <old> → <new> (or "no VERSION file")
```
