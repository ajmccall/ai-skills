---
name: my-design-audit
description: |
  Report-only audit of implemented UI. Reviews live pages or changed UI code for
  hierarchy, spacing, typography, states, responsiveness, and accessibility, then
  writes a design audit report with evidence and recommended fixes. Use when:
  "design audit", "audit the UI", "visual audit", "check the implemented design".
allowed-tools:
  - Bash
  - Read
  - Write
  - AskUserQuestion
  - WebSearch
---

# Design Audit: Report Only

Audit an implemented interface after design and engineering work is already in code.

**This skill is report-only.** It does not fix code, edit plans, or rewrite specs.

## Conventions

- When asking questions: state the context, explain plainly, give a recommendation,
  present lettered options. One decision per question.
- Completion status: DONE | DONE_WITH_CONCERNS | BLOCKED | NEEDS_CONTEXT

## Browser Setup

Detect available browser tooling (check in order):
1. Playwright MCP tools
2. Any other browser automation MCP available in the session
3. If no browser tool is available, work from screenshots, running URLs, or changed
   source context when possible; otherwise stop and tell the user browser tooling is
   required for a full audit

## Scope

Choose the audit target using this order:
1. User-provided URL
2. Running local app
3. Diff-aware UI scope from the current branch

For diff-aware audits:
1. Read the branch diff and identify affected pages/components
2. Map likely routes or entry points
3. Focus the audit on the changed UI plus adjacent flows

## Audit Checklist

For each page or component reviewed, check:

### Hierarchy
- Is the focal point obvious?
- Is there one primary action per view?
- Does the layout communicate what matters first?

### Typography
- Is the type scale coherent?
- Are body copy and headings readable?
- Are line lengths and spacing comfortable?

### Spacing and Layout
- Is the spacing system consistent?
- Do alignment and grouping feel intentional?
- Are dense views still scannable?

### States and Interaction
- Are hover, focus, disabled, empty, loading, and error states present where needed?
- Do interactions feel intentional rather than default?
- Are forms and validation messages understandable?

### Responsiveness and Accessibility
- Does the UI hold up on mobile and desktop?
- Are touch targets usable?
- Is contrast adequate?
- Is keyboard navigation/focus behavior acceptable?

### Design Consistency
- Does this match the established product language?
- Does anything feel generic, sloppy, or visually inconsistent?

## Evidence Rules

1. Every issue should include evidence.
2. Prefer screenshots for visible problems.
3. For interaction issues, capture before/after states when possible.
4. Verify each issue at least twice before recording it.
5. Do not fix anything from this skill.

## Report Output

Write to `.qa-reports/design-audit-{target}-{YYYY-MM-DD}.md`:

```markdown
# Design Audit: {target}
Date: {date} | Pages reviewed: {N}

## Summary
Short overview of overall design quality and biggest issues.

## Top Issues
1. [High] {title} — {one-line impact}

## Findings

### AUDIT-001: {title}
- **Severity:** High / Medium / Low / Polish
- **Category:** Hierarchy / Typography / Spacing / State / Responsive / Accessibility / Consistency
- **Page/Component:** {url or component}
- **Problem:** {what is wrong}
- **Impact:** {why it matters}
- **Evidence:** {screenshot path or observation}
- **Recommended fix:** {concise recommendation}

## Screens Reviewed
| Screen | Status | Issues |
|--------|--------|--------|
```

## Final Output

Report the result in this format:

```text
DESIGN AUDIT SUMMARY
========================
Target:    <url or app>
Pages:     N
Findings:  N

Top issues:
  [HIGH]   <summary>
  [MEDIUM] <summary>
  [POLISH] <summary>

Report: .qa-reports/design-audit-<target>-<date>.md
========================
```

## Rules

- Never fix code from this skill.
- Never rewrite `specs/`, `reviews/`, or `.plans/`.
- Prefer evidence-backed findings over taste-only commentary.
- Focus on implemented UI, not speculative redesign.
