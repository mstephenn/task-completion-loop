# task-completion-loop

A Claude Code plugin providing `/task-completion-loop`: an unattended
branch -> implement -> push -> PR -> automated review -> rework -> merge ->
next-task loop over a repo's `docs/planning/phase-*/sprint-*/task-*.md`
task docs.

## What it does

1. Finds the first not-done task in `docs/planning/`, in phase/sprint/task
   order.
2. Branches, implements it, checks off the acceptance criteria it
   satisfies in the task doc itself.
3. Commits, pushes, and opens a PR/MR on whichever VCS host the repo's
   `origin` remote points to — GitHub, Azure DevOps, or GitLab are
   auto-detected from the remote URL, with org/project/repo identifiers
   parsed from it rather than hardcoded.
4. Runs `/code-review` against the diff as the actual quality gate (no
   human reviewer is assigned). Reworks up to 3 rounds on findings.
5. Merges the PR and repeats, until every task is done or the loop hits a
   stop condition (stuck task, merge failure, or an acceptance criterion
   that requires something outside the loop's control — e.g. human
   sign-off, an external team dependency).

See `commands/task-completion-loop.md` for the exact step-by-step contract
the command follows, including the `skip=<task-id>,...` argument for
carrying known-blocked tasks past future runs without re-triggering a halt
every time.

## Install

```
/plugin marketplace add <this-repo-url>
/plugin install task-completion-loop@task-completion-loop-marketplace
```

## Requirements

- A repo with `docs/planning/phase-*/sprint-*/task-*.md` task docs using
  `- [ ]` / `- [x]` acceptance-criteria checkboxes.
- A CLI matching the repo's `origin` remote host, authenticated:
  - GitHub — `gh`
  - Azure DevOps — `az` with the `azure-devops` extension
  - GitLab — `glab`

  Only the CLI matching the actual remote is needed. An unrecognized or
  unauthenticated host halts the loop before it starts, rather than
  guessing.
- A `/code-review` skill or slash command available in the session (this
  plugin does not ship one — it assumes the target repo or another
  installed plugin provides it).

## Guardrails

- Never commits to or pushes the default branch directly.
- Never force-pushes.
- Rework is capped at 3 rounds per task; exceeding it halts the whole loop,
  not just that task.
- Acceptance criteria requiring human/external action are a hard stop for
  that task — the loop never checks a box it can't actually satisfy.
