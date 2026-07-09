---
description: "Unattended branch -> implement -> push -> PR -> automated review -> rework -> merge -> next-task loop over docs/planning/ task docs"
argument-hint: "[skip=<task-id>,<task-id>,...]"
allowed-tools: ["Bash", "Read", "Write", "Edit", "Grep", "Glob", "Skill"]
---

# Task Completion Loop

Unattended branch -> implement -> push -> PR -> automated review -> rework ->
merge -> next-task loop over `docs/planning/` task docs. Invoked explicitly
only — never auto-triggered — because it pushes branches and merges pull
requests.

**Optional argument:** `skip=<task-id>,<task-id>,...` (e.g. `skip=0.1.3`) —
task IDs to skip over in Step 1's scan, for tasks already known to be
blocked on something outside this loop's control (see Stop Conditions).
Parse `"$ARGUMENTS"` for this before Step 1. Skipped tasks are listed in
the final report; they are never implemented or checked off by this loop.

## Guardrails (read first)

- Never commit to or push the default branch directly.
- Never force-push.
- Never delete branches other than the one just merged for the current task.
- No PR reviewer is set; `/code-review` is the actual quality gate, not a
  human approval.
- Rework is capped at 3 rounds per task. Exceeding the cap halts the
  **entire** loop (not just that task) and reports the stuck task +
  findings to the user.
- A task acceptance criterion the loop cannot satisfy autonomously (e.g.
  "reviewed and explicitly signed off" by a human, an unresolved external
  precondition) is a stop condition for that task: halt and escalate, do
  not check the box and do not skip ahead — unless that task ID was passed
  in `skip=`, in which case move to the next candidate instead.

## Step 0 — Resolve Repo Context

Derive everything about the target repo from the environment; never hardcode
org/project/repo names.

1. `git remote -v` — parse the URL. For an Azure DevOps SSH remote
   (`git@ssh.dev.azure.com:v3/{ORG}/{PROJECT}/{REPO}` or the
   `azure-optisol:v3/{ORG}/{PROJECT}/{REPO}` alias form), `{ORG}`,
   `{PROJECT}`, `{REPO}` are exactly those path segments — case-sensitive,
   do not normalize them.
2. Do **not** trust `az devops configure -l`'s saved defaults — they can
   silently point at the wrong org. Always pass `--organization` and
   `--project` explicitly on every `az repos` call in this loop.
3. Determine the default branch (`main` or `master`) via
   `git symbolic-ref refs/remotes/origin/HEAD` (or default to `main` if
   that's unset and `main` exists on the remote).

## Step 1 — Find the Next Task

1. List all files matching `docs/planning/phase-*/sprint-*/task-*.md`.
2. Sort by phase number, then sprint number, then task number — all
   numeric, ascending (e.g. `phase-2-*` before `phase-10-*`; sort on the
   numbers, not lexicographically on the directory name):
   ```bash
   find docs/planning -path '*/sprint-*/task-*.md' | sort -t/ -k2 -k3 -V
   ```
3. For each file in that order, check every `- [ ]` / `- [x]` line under
   its acceptance-criteria section (`grep -c '^- \[ \]' <task-doc-path>` is
   sufficient — task docs don't nest checkbox lists elsewhere). A task is
   **done** if that count is zero.
4. Skip any task whose `<task-id>` (derived from its filename,
   `task-<task-id>-<task-slug>.md`) appears in the `skip=` argument.
5. The first not-done, not-skipped task in sorted order is
   `<task-doc-path>`. Derive `<task-id>` and `<task-slug>` from its
   filename.
6. Stop conditions for this step:
   - No file matches in step 1, or every matched file is done or skipped:
     **stop the loop** — report "no remaining tasks" (listing anything
     skipped) to the user. This is the success/completion stop condition.

## Step 2 — Branch

```bash
git checkout <default-branch>
git pull origin <default-branch>
git checkout -b task/<task-id>-<task-slug>
```
Never skip the `pull` — branching from a stale local default branch risks
a PR that conflicts or silently drops someone else's merge.

## Step 3 — Plan

Before writing code, write a short plan using the same structure as Plan
Mode: problem/goal, approach, and files you'll touch and why. This plan is
**not** submitted for approval and does not block the loop — write it, then
proceed straight to Step 4. Fold it directly into the PR description
written in Step 5.

Keep it short (3-6 sentences) for small tasks; expand only as far as the
task's acceptance criteria actually require.

## Step 4 — Implement

1. Read `<task-doc-path>` in full — Goal, Context/Why, Implementation Plan
   (if present), and the acceptance-criteria checklist.
2. Implement the task: write the code and tests needed to satisfy every
   unchecked `- [ ]` acceptance criterion.
3. For each acceptance criterion you've satisfied, edit `<task-doc-path>`
   to flip `- [ ]` to `- [x]` for that line. This is the only place "done"
   is recorded — there is no separate status field.
4. If any acceptance criterion requires something the loop cannot do
   itself (external human sign-off, an unresolved precondition owned by
   someone else, an ambiguous spec gap) — **stop the loop** here, leave
   that checkbox unchecked, and report the task + the specific
   unsatisfiable criterion to the user, noting it as a candidate for a
   future `skip=` argument. Do not check it off speculatively and do not
   move to the next task while this one is incomplete.

## Step 5 — Commit, Push, Open the PR

```bash
git add -A
git commit -m "feat: implement task <task-id> - <short description>"
git push -u origin task/<task-id>-<task-slug>
```

Using the org/project/repo resolved in Step 0:
```bash
az repos pr create \
  --organization https://dev.azure.com/<ORG> \
  --project <PROJECT> \
  --repository <REPO> \
  --source-branch task/<task-id>-<task-slug> \
  --target-branch <default-branch> \
  --title "Task <task-id>: <short description>" \
  --description "<task plan from Step 3, or a one-line summary for trivial tasks>" \
  --output json
```
Capture the returned `pullRequestId` as `<pr-id>`. `--project` is required
even though a default may be configured — passing `--organization`
explicitly suppresses the default lookup, so always pass both.

No reviewer is added to the PR — Step 6 below is the actual gate.

## Step 6 — Automated Review and Rework

1. Run the `/code-review` skill against the diff on
   `task/<task-id>-<task-slug>` (the PR's source branch).
2. If `/code-review` reports no blocking findings: proceed to Step 7.
3. If it reports findings: fix them, commit, `git push` (same branch — the
   open PR updates automatically), and re-run `/code-review`. This is one
   "rework round."
4. Track rework rounds for this task. After 3 rework rounds without a
   clean review: **stop the entire loop** (not just this task). Report the
   task, the PR, and the latest `/code-review` findings to the user. Do
   not proceed to Step 7 and do not move to another task.

## Step 7 — Merge

```bash
az repos pr update \
  --organization https://dev.azure.com/<ORG> \
  --id <pr-id> \
  --status completed \
  --delete-source-branch true \
  --merge-commit-message "Merge task <task-id>: <short description>"
```
Note: `az repos pr update` does **not** accept `--project` — passing it
errors. This is inconsistent with `pr create` but is just how the CLI
behaves.

Merge completion is **queued, not synchronous**. Poll before treating it as
done:
```bash
for i in 1 2 3 4 5; do
  sleep 2
  az repos pr show --organization https://dev.azure.com/<ORG> --id <pr-id> \
    --query "{status:status, mergeStatus:mergeStatus}" -o json
done
```
Wait for `"status": "completed"` and `"mergeStatus": "succeeded"` before
moving on. If it comes back `"mergeStatus": "failed"` or `"conflicts"`,
treat this the same as a rework-cap failure: stop the loop and report to
the user — do not retry merge indefinitely.

```bash
git checkout <default-branch>
git pull origin <default-branch>
git branch -d task/<task-id>-<task-slug>
```
(The remote branch is already gone — `--delete-source-branch true` handled
that. This is local cleanup only.)

## Step 8 — Repeat

Go back to **Step 1 — Find the Next Task**. Continue the cycle until one of
the Stop Conditions below is hit.

## Stop Conditions

- **Success:** Step 1 finds no remaining undone, unskipped task anywhere
  under `docs/planning/`. Report completion to the user with a summary of
  every task processed this run and every task skipped via `skip=`.
- **Stuck task:** a task exceeds 3 rework rounds (Step 6) or its merge
  fails (Step 7). Halt the entire loop immediately — do not start another
  task. Report the stuck task, its PR, and the relevant findings/error to
  the user.
- **Unsatisfiable acceptance criterion:** Step 4 finds a checkbox the loop
  cannot resolve autonomously. Halt the entire loop immediately. Report
  the task and the specific criterion to the user, and suggest re-running
  with `skip=<task-id>` if the user wants the loop to continue past it.
