# AI Skills

This repo contains a set of reusable Codex skills that together form a practical product and engineering workflow.

At a high level, the skills cover:

- thinking: shape ideas, challenge scope, and turn vague requests into a concrete spec
- planning: review the spec, pressure-test the approach, and produce an implementation plan
- execution: implement work, review the diff, ship it, and land it
- debugging and quality: investigate bugs from root cause, QA flows, and audit design or security
- reflection: look back on what shipped and how the team worked

These skills are designed to be composable rather than mandatory. You do not need to use every skill on every task. The value is in chaining the right ones together for the job.

## Common Workflows

### Feature Workflow

For new product or engineering work, the usual flow is:

`my-ideate` -> `my-eng-review` -> `my-plan` -> `my-implement` -> `my-review` -> `my-ship` -> `my-land`

- `my-ideate`: turns an idea into a sharper spec
- `my-eng-review`: challenges architecture, scope, and test coverage before code
- `my-plan`: writes a concrete implementation plan
- `my-implement`: executes the plan task by task
- `my-review`: reviews the branch diff for structural issues before shipping
- `my-ship`: prepares the branch, tests it, pushes, and opens a PR
- `my-land`: merges, watches deploys, and verifies production health

### Debug Workflow

For bugs, start with root cause, not a patch:

`my-investigate` -> `my-review` -> `my-ship` -> `my-land`

- `my-investigate`: reproduces the issue, finds the real cause, implements the fix, and adds a regression test
- `my-review`: checks the fix for hidden risks, scope drift, and test gaps
- `my-ship` / `my-land`: handle the normal path to PR, merge, and deploy

If the bug is user-facing in the browser, `my-qa` can be used before or after the fix to verify the affected flow.

### Design Workflow

For UX or visual work, there are two common entry points:

`my-ideate` -> `my-design-review` -> `my-plan` -> `my-implement`

or, for an already built UI:

`my-design-review` -> `my-qa`

- `my-design-review` can review a plan before implementation or audit live UI code after implementation
- `my-qa` tests the product like a user and produces a report with evidence, without making changes

## Supporting Skills

- `my-security`: security posture review for a branch or full repo
- `my-retro`: weekly retrospective on commits, quality, and team patterns

Use these when the task needs extra scrutiny, not as mandatory steps in every workflow.
