---
name: my-land
description: |
  Land and deploy. Merges PR, waits for CI and deploy, verifies production
  health, offers revert on failure. Picks up after /my-ship creates the PR.
  Use when: "merge", "land", "deploy", "land it", "ship to production".
allowed-tools:
  - Bash
  - Read
  - Write
  - Glob
  - AskUserQuestion
---

# Land & Deploy: Merge, Deploy, Verify

Mostly automated. The user said `/my-land` which means DO IT.

## Conventions

- When asking questions: state the context (project, branch, task), explain plainly,
  give a recommendation, present lettered options. One decision per question.
- Completion status: DONE | DONE_WITH_CONCERNS | BLOCKED | NEEDS_CONTEXT

**Always stop for:**
- Pre-merge readiness gate (Step 3)
- No PR found, CI failures, merge conflicts, permission denied
- Deploy failure or production health issues (offer revert)

**Never stop for:**
- Choosing merge method
- Timeout warnings (warn and continue)

---

## Step 0: Detect base branch

1. `gh pr view --json baseRefName -q .baseRefName`
2. Fallback: `gh repo view --json defaultBranchRef -q .defaultBranchRef.name`
3. Fallback: `main`

---

## Step 1: Pre-flight

1. Check GitHub CLI: `gh auth status`. If not authenticated, **STOP.**
2. Parse arguments: `/my-land` (auto-detect PR), `/my-land #123` (specific PR),
   `/my-land <url>` (auto-detect PR, verify at URL).
3. Detect PR: `gh pr view --json number,state,title,url,mergeable,baseRefName,headRefName`
4. Validate:
   - No PR → **STOP.** "Run /my-ship first."
   - Already merged → capture merge metadata, skip Step 4, and continue to deploy
     detection / final reporting so the plan record is still updated.
   - Closed → "Reopen it first."
   - Open → continue.
5. Find the active plan in `.plans/`.
   Read its `Execution Status`, `Task Checklist`, `Decisions Log`, and `Outcomes / Drift`
   so merge/deploy reporting updates the same execution record.
6. Read the plan's `## Coordination` block if present so branch ownership and repo
   split stay accurate after merge.

---

## Step 2: Pre-merge checks

```bash
gh pr checks --json name,state,status,conclusion
```

- Required checks FAILING → **STOP.** Show failures.
- Required checks PENDING → go to Step 2.5.
- All pass → skip to Step 3.

Check for conflicts: `gh pr view --json mergeable -q .mergeable`
If `CONFLICTING` → **STOP.** "Resolve merge conflicts first."

---

## Step 2.5: Wait for CI (if pending)

```bash
gh pr checks --watch --fail-fast
```

Timeout: 15 minutes. If passes → continue. If fails → **STOP.** If timeout → **STOP.**

---

## Step 3: Pre-merge readiness gate

Run the project's test command (detect from CLAUDE.md or auto-detect, same as /my-ship).

Present a readiness summary via AskUserQuestion:

- PR number, title, branch → base
- Tests: PASS / FAIL
- CI checks: PASS / PENDING / FAIL
- Merge conflicts: NONE / CONFLICTING

If tests fail → recommend "Don't merge yet."

Options:
- A) Merge — checks passed
- B) Don't merge — address issues first
- C) Merge anyway — I understand the risks

If B → **STOP.** List what needs fixing.
If A or C → continue.

---

## Step 4: Merge

Always squash merge:
```bash
gh pr merge --squash --delete-branch
```

If permission error → **STOP.**

If merge queue active, poll `gh pr view --json state -q .state` every 30s, up to 30min.

Capture merge commit SHA and timestamp.

---

## Step 5: Deploy detection

Detect deploy strategy:

```bash
# Platform detection
[ -f fly.toml ] && echo "PLATFORM:fly"
[ -f render.yaml ] && echo "PLATFORM:render"
([ -f vercel.json ] || [ -d .vercel ]) && echo "PLATFORM:vercel"
[ -f netlify.toml ] && echo "PLATFORM:netlify"
[ -f Procfile ] && echo "PLATFORM:heroku"

# GitHub Actions deploy workflows
for f in .github/workflows/*.yml .github/workflows/*.yaml; do
  [ -f "$f" ] && grep -qiE "deploy|release|production|cd" "$f" 2>/dev/null && echo "DEPLOY_WORKFLOW:$f"
done
```

**Decision tree:**
1. User provided URL → use for canary, also check for deploy workflows.
2. Deploy workflow found → poll in Step 6, then canary.
3. No deploy workflow, no URL → AskUserQuestion:
   - A) Provide a production URL to verify
   - B) Skip verification — no web deploy

---

## Step 6: Wait for deploy (if applicable)

**GitHub Actions:** Find the run triggered by merge commit, poll every 30s:
```bash
gh run list --branch <base> --limit 10 --json databaseId,headSha,status,conclusion
gh run view <run-id> --json status,conclusion
```

**Auto-deploy platforms (Vercel, Netlify):** Wait 60s, then canary.

**Fly/Render/Heroku:** Check platform CLI if available, otherwise poll URL.

If deploy fails → AskUserQuestion:
- A) Investigate deploy logs
- B) Create a revert commit
- C) Continue anyway

Timeout 20min → warn and ask whether to continue or skip.

---

## Step 7: Canary verification

Verify production health. Use available browser tooling if present (Playwright MCP,
or similar). Fall back to `curl` for basic HTTP checks.

**4 checks:**
1. Page loads successfully (HTTP 200)
2. No critical console errors (if browser tool available)
3. Page has real content (not blank or error page)
4. Loads in under 10 seconds

If all pass → HEALTHY.

If any fail → AskUserQuestion with evidence:
- A) Expected — deploy propagating, mark healthy
- B) Broken — create revert commit
- C) Investigate further

---

## Step 8: Revert (if needed)

```bash
git fetch origin <base> && git checkout <base>
git revert <merge-commit-sha> --no-edit
git push origin <base>
```

If conflicts → warn, suggest manual revert.
If branch protections → suggest revert PR instead.

---

## Step 9: Deploy report

```
LAND & DEPLOY REPORT
═════════════════════
PR:           #<number> — <title>
Branch:       <head> → <base>
Merged:       <timestamp>
Merge SHA:    <sha>

Timing:
  CI wait:    <duration>
  Deploy:     <duration or "no workflow detected">
  Canary:     <duration or "skipped">
  Total:      <end-to-end>

CI:           PASSED / SKIPPED
Deploy:       PASSED / FAILED / NO WORKFLOW
Verification: HEALTHY / DEGRADED / SKIPPED / REVERTED

VERDICT: DEPLOYED AND VERIFIED / DEPLOYED (UNVERIFIED) / REVERTED
```

## Step 9.5: Update Plan Record

If a matching plan exists, update it before finishing:

1. Set `Execution Status` to:
   - `Merged` after the PR lands
   - `Deployed` if deploy verification also completed successfully
   - `Reverted` if Step 8 is used
2. Update the `## Coordination` block with the final merged branch and PR state
   if the plan still tracks active branches
3. Append merge method (`squash`), merge SHA, PR URL, and deployment verification
   outcome to `Outcomes / Drift`
4. Append any land-time decisions or exceptions to `Decisions Log`

The plan should end the workflow as a concise, accurate execution ledger.
