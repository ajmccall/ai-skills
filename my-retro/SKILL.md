---
name: my-retro
description: |
  Weekly engineering retrospective. Analyzes commit history, work patterns, and
  code quality metrics. Team-aware with per-person breakdown. Persistent history
  with trend tracking. Use when: "retro", "what did we ship", "weekly retro".
allowed-tools:
  - Bash
  - Read
  - Write
  - Glob
  - AskUserQuestion
---

# Engineering Retrospective

Analyze commit history and work patterns for a time window. Produces a retro report
with metrics, trends, and per-contributor analysis.

## Conventions

- Completion status: DONE | DONE_WITH_CONCERNS | BLOCKED | NEEDS_CONTEXT

## Arguments

- `/my-retro` — last 7 days (default)
- `/my-retro 24h` — last 24 hours
- `/my-retro 14d` — last 14 days
- `/my-retro 30d` — last 30 days
- `/my-retro compare` — this period vs prior same-length period

---

## Step 1: Setup

1. Parse argument for time window (default 7 days).
2. For day/week units, compute absolute start date at local midnight:
   e.g., 7d window on 2026-03-30 = `--since="2026-03-23T00:00:00"`
3. Detect default branch:
   ```bash
   gh repo view --json defaultBranchRef -q .defaultBranchRef.name 2>/dev/null || echo main
   ```
4. Identify current user: `git config user.name`

---

## Step 2: Gather Data

Run these git commands (all independent, run in parallel):

```bash
# Commits with metadata
git log origin/<default> --since="<window>" --format="%H|%aN|%ae|%ai|%s" --shortstat

# Per-commit file stats (test vs prod)
git log origin/<default> --since="<window>" --format="COMMIT:%H|%aN" --numstat

# Timestamps for session detection
git log origin/<default> --since="<window>" --format="%at|%aN|%ai|%s" | sort -n

# File hotspots
git log origin/<default> --since="<window>" --format="" --name-only | grep -v '^$' | sort | uniq -c | sort -rn | head -15

# Per-author commit counts
git shortlog origin/<default> --since="<window>" -sn --no-merges

# TODOS.md if exists
cat TODOS.md 2>/dev/null || true
```

---

## Step 3: Compute Metrics

| Metric | Value |
|--------|-------|
| Commits to default branch | N |
| Contributors | N |
| Total insertions | N |
| Total deletions | N |
| Net LOC | N |
| Test LOC ratio | N% |
| Active days | N |
| Detected sessions | N |

**Per-author leaderboard:**

```
Contributor         Commits   +/-          Top area
You (name)               N   +N/-N        dir/
teammate                 N   +N/-N        dir/
```

Current user appears first, labeled "You (name)".

---

## Step 4: Commit Time Distribution

Hourly histogram in local time:

```
Hour  Commits  ████████████████
 09:    8      ████████
 10:   12      ████████████
 ...
```

Call out: peak hours, dead zones, late-night clusters.

---

## Step 5: Work Session Detection

Use 45-minute gap threshold between consecutive commits.

Classify:
- **Deep** (50+ min)
- **Medium** (20-50 min)
- **Micro** (<20 min)

Calculate: total active time, average session length, LOC per active hour.

---

## Step 6: Commit Type Breakdown

Categorize by conventional commit prefix (feat/fix/refactor/test/chore/docs):

```
feat:     20  (40%)  ████████████████████
fix:      27  (54%)  ███████████████████████████
refactor:  2  ( 4%)  ██
```

Flag if fix ratio >50% — may indicate review gaps.

---

## Step 7: Hotspot Analysis

Top 10 most-changed files. Flag:
- Files changed 5+ times (churn hotspots)
- Test vs production files in the list

---

## Step 8: Per-Contributor Analysis

For each contributor:
1. Commits and LOC
2. Top 3 focus areas (directories)
3. Commit type mix
4. Session patterns and peak hours
5. Test discipline (their test LOC ratio)
6. Biggest ship (highest-impact commit/PR)

For the current user: deepest treatment, framed as "your".
For teammates: 2-3 sentences + one praise + one growth opportunity (anchored in data).

**Co-authored commits:** Parse `Co-Authored-By:` trailers. Credit both authors.
Note AI co-authors separately as "AI-assisted commits" metric.

---

## Step 9: Shipping Streak

```bash
# Team streak
git log origin/<default> --format="%ad" --date=format:"%Y-%m-%d" | sort -u
# Personal streak
git log origin/<default> --author="<user>" --format="%ad" --date=format:"%Y-%m-%d" | sort -u
```

Count consecutive days with commits backward from today.

---

## Step 10: History & Trends

```bash
ls -t .context/retros/*.json 2>/dev/null | head -1
```

If prior retro exists, load it and show deltas:

```
                Last     Now      Delta
Test ratio:     22%  ->  41%      +19pp
Sessions:       10   ->  14       +4
Fix ratio:      54%  ->  30%      -24pp (improving)
```

If first retro: "First retro recorded — run again next week to see trends."

---

## Step 11: Save & Report

Save JSON snapshot:
```bash
mkdir -p .context/retros
```

Write `retro-YYYY-MM-DD.json` with all computed metrics.

Write the retro report to stdout as markdown. If `compare` mode, include
side-by-side period comparison.

---

## Rules

- All times in user's local timezone.
- Do NOT make code changes.
- Praise must be anchored in specific commits, not generic.
- Growth suggestions framed as leveling-up, not criticism.
