---
name: taskloop-plan
description: Use when the user asks Codex to plan taskloop backlog items, decompose a goal or spec into docs/planning, Jira, Asana, Monday.com, or Linear tasks, or invoke taskloop-plan with a goal, spec path, or source= argument.
---

# Task Loop Plan

Use this skill when the user wants to create backlog tasks for Task Loop. This
skill creates tasks only; it does not branch, push, open PRs/MRs, or merge.

## Required References

Before taking task actions, read these files in full:

1. `../../commands/taskloop-plan.md`
2. `../../commands/taskloop.md`

Follow `taskloop-plan.md` as the authoritative planning workflow. Use
`taskloop.md` only for the shared task-source adapter details in Appendices B
and C.

## Codex Mapping

- Treat user text after "taskloop-plan" as the command arguments described by
  the reference, including a goal/spec path and `source=`.
- Use Codex shell/file tools in place of Claude `Bash`, `Read`, `Write`,
  `Edit`, `Grep`, and `Glob` tool names.
- When the reference requires confirmation before creating tasks, stop and ask
  the user. Do not create tasks before explicit acceptance.
- For the `docs` adapter, create files under `docs/planning/` using the exact
  structure described by the reference.

## Reporting

Report created task IDs, already-covered items, and any first-time
`docs/planning/` tracking decision.
