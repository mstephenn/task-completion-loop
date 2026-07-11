---
name: taskloop
description: Use when the user asks Codex to run the taskloop workflow, complete backlog tasks unattended, process docs/planning or Jira/Asana/Monday/Linear tasks through branch, implementation, PR/MR, automated review, merge, and repeat, or use taskloop arguments such as skip=, unblock=, or source=.
---

# Task Loop

Use this skill only when explicitly requested. The workflow can push branches,
open PRs/MRs, merge them, and mark tasks done.

## Required Reference

Before taking task actions, read `../../commands/taskloop.md` in full and follow
it as the authoritative contract.

## Codex Mapping

- Treat user text after "taskloop" as the command arguments described by the
  reference, including `skip=`, `unblock=`, and `source=`.
- Use Codex shell/file tools in place of Claude `Bash`, `Read`, `Write`,
  `Edit`, `Grep`, and `Glob` tool names.
- Use Codex's normal planning behavior in place of Claude "Plan Mode" wording,
  but do not pause for approval unless the reference says to ask the user.
- For the automated review gate, use the best available Codex code-review
  capability in the session. If no review skill/plugin/command is available,
  stop before merging and report that the review gate is unavailable.
- Preserve the reference guardrails: never push directly to the default branch,
  never force-push, never expose tokens, and stop on unresolved adapter or merge
  failures.

## Reporting

Keep progress updates concise and report each processed, skipped, blocked, or
stuck task at the end of the run.
