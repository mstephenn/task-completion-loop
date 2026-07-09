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

## Step 0 — Resolve Repo Context and VCS Provider

Derive everything about the target repo and its hosting provider from the
environment; never hardcode an org/project/repo name or assume one
provider.

1. Run `git remote get-url origin` and match the host in the URL against
   the table below to pick `<provider>`. Do this once per loop run and
   reuse the result — do not re-detect per task.

   | Host pattern in remote URL | `<provider>` | CLI |
   |---|---|---|
   | `github.com` | `github` | `gh` |
   | `dev.azure.com`, `ssh.dev.azure.com`, `visualstudio.com`, or an alias resolving to one of these (check `git config --get remote.origin.url` and any `url.<base>.insteadOf` rewrites in `.gitconfig`) | `azuredevops` | `az` (with the `azure-devops` extension) |
   | `gitlab.com` or a self-hosted GitLab (verify with `glab repo view` succeeding) | `gitlab` | `glab` |
   | anything else | unrecognized | none |

   If unrecognized: **stop the loop** before Step 1 and report the remote
   URL to the user — ask which CLI/workflow to use rather than guessing.

2. Confirm the CLI is installed and authenticated before doing any work:
   - `github`: `gh auth status`
   - `azuredevops`: `az extension show --name azure-devops` (run
     `az extension add --name azure-devops` if missing), then a lightweight
     auth check such as `az repos list --organization <ORG>` for the org
     resolved below.
   - `gitlab`: `glab auth status`

   If not authenticated, stop the loop and tell the user which CLI needs
   login — do not attempt to authenticate on their behalf.

3. Parse `<ORG>`/`<PROJECT>`/`<REPO>` (or the provider's equivalent
   identifiers) from the remote URL itself, per provider:
   - `azuredevops`: `git@ssh.dev.azure.com:v3/{ORG}/{PROJECT}/{REPO}` (or
     an `insteadOf`-aliased form of the same path) — `{ORG}`, `{PROJECT}`,
     `{REPO}` are exactly those path segments, case-sensitive, do not
     normalize them. Do **not** trust `az devops configure -l`'s saved
     defaults — they can silently point at the wrong org. Always pass
     `--organization` and `--project` explicitly on every `az repos` call.
   - `github`: `{OWNER}/{REPO}` from `github.com/{OWNER}/{REPO}` (or the
     SSH equivalent). `gh` infers this from the current directory's remote
     automatically, so explicit `--repo {OWNER}/{REPO}` flags are optional
     but harmless to include for clarity.
   - `gitlab`: `{NAMESPACE}/{REPO}` (may include subgroups,
     `{GROUP}/{SUBGROUP}/{REPO}`) from the remote URL. `glab` also infers
     this from the current directory by default.

4. Determine the default branch (`main` or `master`) via
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

Open the PR using the `<provider>` resolved in Step 0:

**`github`:**
```bash
gh pr create \
  --title "Task <task-id>: <short description>" \
  --body "<task plan from Step 3, or a one-line summary for trivial tasks>" \
  --base <default-branch> \
  --head task/<task-id>-<task-slug>
```
Capture the printed PR URL (or re-run with `--json url,number` for a
machine-readable form) as `<pr-ref>`.

**`azuredevops`:**
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
Capture the returned `pullRequestId` as `<pr-ref>`. `--project` is required
even though a default may be configured — passing `--organization`
explicitly suppresses the default lookup, so always pass both.

**`gitlab`:**
```bash
glab mr create \
  --title "Task <task-id>: <short description>" \
  --description "<task plan from Step 3, or a one-line summary for trivial tasks>" \
  --source-branch task/<task-id>-<task-slug> \
  --target-branch <default-branch> \
  --yes
```
Capture the printed MR URL/IID as `<pr-ref>`.

No reviewer is added to the PR/MR — Step 6 below is the actual gate.

## Step 6 — Automated Review and Rework

1. Run the `/code-review` skill against the diff on
   `task/<task-id>-<task-slug>` (the PR's source branch). This step is
   identical regardless of `<provider>` — it operates on the local git
   diff, not the hosted PR.
2. If `/code-review` reports no blocking findings: proceed to Step 7.
3. If it reports findings: fix them, commit, `git push` (same branch — the
   open PR/MR updates automatically on every provider), and re-run
   `/code-review`. This is one "rework round."
4. Track rework rounds for this task. After 3 rework rounds without a
   clean review: **stop the entire loop** (not just this task). Report the
   task, the PR/MR, and the latest `/code-review` findings to the user. Do
   not proceed to Step 7 and do not move to another task.

## Step 7 — Merge

Merge using `<pr-ref>` and `<provider>` from Steps 0 and 5.

**`github`:**
```bash
gh pr merge <pr-ref> --squash --delete-branch
```
(Use `--merge` instead of `--squash` if the repo's convention — check
recent merge commits with `git log --merges -5` — favors merge commits
over squashing.) `gh pr merge` is synchronous; a nonzero exit code or a
"not mergeable" message means treat this like a rework-cap failure — stop
the loop and report to the user, do not retry indefinitely.

**`azuredevops`:**
```bash
az repos pr update \
  --organization https://dev.azure.com/<ORG> \
  --id <pr-ref> \
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
  az repos pr show --organization https://dev.azure.com/<ORG> --id <pr-ref> \
    --query "{status:status, mergeStatus:mergeStatus}" -o json
done
```
Wait for `"status": "completed"` and `"mergeStatus": "succeeded"` before
moving on. If it comes back `"mergeStatus": "failed"` or `"conflicts"`,
stop the loop and report to the user — do not retry merge indefinitely.

**`gitlab`:**
```bash
glab mr merge <pr-ref> --squash --remove-source-branch --yes
```
`glab mr merge` is synchronous; a nonzero exit code or a pipeline-blocked
message means stop the loop and report to the user rather than retrying.

Then, for every provider:
```bash
git checkout <default-branch>
git pull origin <default-branch>
git branch -d task/<task-id>-<task-slug>
```
(The remote branch is already gone — the merge command above handled
deleting it on every provider. This is local cleanup only.)

## Step 8 — Repeat

Go back to **Step 1 — Find the Next Task**. Continue the cycle until one of
the Stop Conditions below is hit.

## Stop Conditions

- **Success:** Step 1 finds no remaining undone, unskipped task anywhere
  under `docs/planning/`. Report completion to the user with a summary of
  every task processed this run and every task skipped via `skip=`.
- **Unrecognized or unauthenticated VCS provider:** Step 0 can't match the
  remote to a supported provider, or the matching CLI isn't authenticated.
  Halt before Step 1 and ask the user how to proceed.
- **Stuck task:** a task exceeds 3 rework rounds (Step 6) or its merge
  fails (Step 7). Halt the entire loop immediately — do not start another
  task. Report the stuck task, its PR/MR, and the relevant findings/error
  to the user.
- **Unsatisfiable acceptance criterion:** Step 4 finds a checkbox the loop
  cannot resolve autonomously. Halt the entire loop immediately. Report
  the task and the specific criterion to the user, and suggest re-running
  with `skip=<task-id>` if the user wants the loop to continue past it.
