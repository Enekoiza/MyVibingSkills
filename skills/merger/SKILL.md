---
name: merger
description: Merges the open pull requests of the current repository that are ready (no conflicts, green checks, not draft) and deletes branches whose code is already merged into the main branch (master/main). Invoke manually with /merger.
argument-hint: "[merge|squash|rebase]"
model: claude-sonnet-5-5
effort: medium
disable-model-invocation: true
allowed-tools: Bash(gh auth status:*), Bash(gh repo view:*), Bash(gh pr list:*), Bash(gh pr view:*), Bash(gh pr checks:*), Bash(git fetch:*), Bash(git branch -r *), Bash(git branch --show-current*), Bash(git merge-base --is-ancestor *), Bash(git status:*), Bash(git rev-parse:*), Bash(git ls-remote:*), Bash(git pull --ff-only:*)
---

# Merger

Merges the open PRs that are ready and cleans up branches whose code is already in the main branch. Always works on the repository in the current directory and uses `gh` (GitHub CLI).

Always talk to the user in English.

Merging and deleting branches are hard to undo, so this skill first analyses, then shows a plan, and only acts once the user confirms.

## 1. Pre-flight checks

1. `gh auth status` — if there is no session, stop and tell the user to run `gh auth login`.
2. `git rev-parse --is-inside-work-tree` — if it is not a git repo, stop.
3. `git fetch --prune origin` to bring the remote state up to date.
4. Detect the main branch:
   `gh repo view --json defaultBranchRef,mergeCommitAllowed,squashMergeAllowed,rebaseMergeAllowed`
   It will usually be `master`. From here on it is called the **main branch**.

If any command in this section exits non-zero, stop and tell the user what failed.

## 2. Merge method

- If the user passed an argument (`$ARGUMENTS` = `merge`, `squash` or `rebase`), use that.
- Otherwise, use the first one the repo allows in this order: `merge`, `squash`, `rebase`.
- If the requested method is not allowed in the repo, say so and use an allowed one.

## 3. Analyse the open PRs

```bash
gh pr list --state open --limit 100 \
  --json number,title,author,headRefName,headRefOid,baseRefName,isDraft,mergeable,mergeStateStatus,reviewDecision,statusCheckRollup,headRepositoryOwner,isCrossRepository,createdAt
```

Record each PR's `headRefOid`: it is the head SHA shown in the plan and the only commit that may be merged.

Classify each PR:

| State | Condition | Action |
|---|---|---|
| ✅ Ready | not draft, `mergeable = MERGEABLE`, `mergeStateStatus` is `CLEAN` or `HAS_HOOKS` | Merge |
| ⏳ Pending checks | any check in `PENDING`/`IN_PROGRESS`/`QUEUED` | Skip |
| ❌ Failing checks | any check in `FAILURE`/`ERROR`/`CANCELLED`/`TIMED_OUT` | Skip |
| ⚠️ Conflicts | `mergeable = CONFLICTING` or `mergeStateStatus = DIRTY` | Skip |
| 🔄 Out of date | `mergeStateStatus = BEHIND` | Skip (say it needs updating with the base branch) |
| 🔒 Blocked | `mergeStateStatus = BLOCKED` or `reviewDecision = CHANGES_REQUESTED` / `REVIEW_REQUIRED` | Skip |
| 📝 Draft | `isDraft = true` | Skip |
| ❓ Unknown | `mergeable = UNKNOWN` | Query once more with `gh pr view <n> --json mergeable,mergeStateStatus` (GitHub computes it in the background); if still unknown, skip |

Never force a merge with `--admin` or by bypassing branch protections.

## 4. Find already-merged branches (no open PR)

Besides the open PRs, find remote branches whose code is already in the main branch but which still exist:

```bash
gh pr list --state merged --base <main branch> --limit 100 --json number,headRefName,headRefOid,isCrossRepository,mergedAt
```

Group the merged PRs by `headRefName` (one branch can have several), ignoring PRs with `isCrossRepository = true` and branches protected by section 5, and run this check per candidate branch. This also covers squash merges, which `git branch --merged` does not detect.

- (a) Run `git ls-remote origin refs/heads/<branch>`. Everywhere in this skill, "returns nothing" means it exited 0 with empty output: the branch is already gone. A non-zero exit is always an error, never 'already gone'. Here, a branch that returns nothing is dropped and not reported; a non-zero exit is a step-4 error: plan no delete for that branch.
- (b) Take the full 40-character SHA it returns as the **current tip**.
- (c) The branch is deletable only if one of these holds, checked in this order:
  - first, the tip equals the `headRefOid` of any merged PR from that branch (this covers squash and rebase merges); if so, the branch is deletable and `merge-base` is not run;
  - only otherwise, `git merge-base --is-ancestor <tip> origin/<main branch>` exits 0. Exit code 1 means "not an ancestor"; it is not an error. Any other exit code (e.g. 128 when the tip object is missing locally) is an error, never 'kept': do not plan a delete for that branch and report it under errors.
- (d) Otherwise, if neither condition holds and nothing in (a) or (c) was an error, classify the branch as **"has commits after merge — kept"**. Never delete it.

`git branch -r --merged origin/<main branch>` is only a cross-check for regular merge commits: a branch it lists that has no merged PR above is not a candidate and is never deleted.

## 5. Branches that are NEVER deleted

- The main branch and `master`, `main`, `develop`, `dev`, `release/*`, `hotfix/*` when they are long-lived branches of the repo.
- Any branch that is the **base** of another open PR (deleting it would close that PR).
- Any branch that is the **head** of another open PR.
- Fork branches (`isCrossRepository = true`): they are not ours, do not touch them.
- The local branch the user is currently on (`git branch --show-current`), while it is checked out: its remote branch may be deleted in step 7, and its local copy only in step 8.5, after switching away.

Only delete branches of PRs whose base is the **main branch**. If a PR targets another branch (for example, a feature going into `develop`), it is merged but its branch is kept, because its code is not yet in the main branch.

## 6. Show the plan and ask for confirmation

Show a compact summary before doing anything:

```
Repo: <owner>/<repo> · main branch: master · method: merge

To be merged (N):
  #12  Add export endpoint        feature/export @a1b2c3d → master  (branch will be deleted)
  #15  Fix pagination             fix/paging @b2c3d4e     → develop (branch is kept)
  #17  Update README              docs/readme @e5f6a7b    → master  (branch will be deleted)

Already-merged branches to delete (M):
  feature/login  @c3d4e5f  (PR #9, merged on 2026-09-10)

Kept — commits after merge (J):
  fix/old-paging  @d4e5f6a  (PR #7 merged @9f8e7d6; tip differs)

Errors (E):
  feature/old  ls-remote failed (exit 128) — no delete planned

Skipped (K):
  #14  ❌ failing checks (build)
  #16  ⚠️ conflicts with master
```

Ask: "Shall I proceed with everything, exclude any, or cancel?". Do not continue without an explicit yes. Excluded items are not touched. If there is nothing to merge or delete, show the plan without the question and finish.

## 7. Execute

`<headRefOid>` is the head SHA recorded for the PR in step 3; `<plan sha>` is the full 40-character current tip recorded for the branch in step 4 (b). The plan shows both shortened, for display only. Classify every outcome with the table in section 9; only an **error** in this step stops step 8 from switching branch. Never retry a failed command.

Merge the PRs in creation order (oldest first). For each PR:

1. Query its state again, because merging one PR can cause conflicts or leave the following ones `BEHIND`: `gh pr view <n> --json mergeable,mergeStateStatus,headRefOid`. If it is no longer ready, or its `headRefOid` differs from `<headRefOid>`, skip it. Never merge a head the user did not see.
2. `gh pr merge <n> --<method> --match-head-commit <headRefOid>`. Never pass `--delete-branch` (gh deletes the local branch with `git branch -D`). If it exits non-zero, the PR was not merged: do not touch its branch, and run `gh pr view <n> --json headRefOid` once to tell a changed head from another failure.
3. Only if the PR targets the main branch and its branch is not protected by section 5: run `git ls-remote origin refs/heads/<branch>`; only if it returns `<headRefOid>`, **lease delete** the branch with `<expected sha>` = `<headRefOid>`. PRs to another base and protected branches: leave the branch alone.

Then, for each already-merged branch from step 4, immediately before deleting it run `git ls-remote origin refs/heads/<branch>`; only if it returns `<plan sha>`, **lease delete** it with `<expected sha>` = `<plan sha>`.

**Lease delete.** `git push --force-with-lease=refs/heads/<branch>:<expected sha> origin --delete <branch>`. Never add `--force`. If it exits non-zero for any reason (stale-info rejection, `remote ref does not exist`, or anything else), the remote delete did not happen: run `git ls-remote origin refs/heads/<branch>` exactly once and classify the result with section 9.

**Local delete.** Only after a successful remote delete, and only if the branch exists locally (`git rev-parse --verify --quiet refs/heads/<branch>` exits 0). If it is the current branch, leave it for step 8.5. Otherwise run `git branch -d <branch>`. Never use `-D`, not even after `-d` refuses an unmerged branch (typical after a squash).

## 8. Leave the repo on the main branch

If there was any error in step 7, do not switch branch: only run `git fetch --prune origin` and leave the user where they were so they can review. Otherwise (step-4 errors, skips and kept branches do not count):

1. `git status --porcelain` — if there are uncommitted changes, **do not switch branch**; go to the summary.
2. `git switch <main branch>` — if it fails, go to the summary.
3. `git pull --ff-only origin <main branch>` — if it fails (e.g. the local main branch has commits that are not on origin), force nothing, leave the branch as it is and continue.
4. `git fetch --prune origin`
5. If step 7 left the local copy of the branch you started on only because it was the current branch (its remote was deleted), delete it now with `git branch -d <branch>` (never `-D`).

If step 8 stopped before 8.5 (errors in step 7, 8.1 or 8.2), that local copy stays.

## 9. Final summary

```
✅ Merged: #12, #15
🗑️  Deleted branches (remote; local too where it existed, unless listed under Left locally or Errors): feature/export, feature/login
🛡️  Kept (commits after merge / changed since plan): fix/old-paging
ℹ️  Already gone (remote): <branches or none>
💻 Left locally: <branch (-d refused) | branch (current branch, not switched) — or none>
⏭️  Skipped: #14 (failing checks), #16 (conflicts), #17 (head changed since plan)
⚠️  Errors: <#n or branch (step 4|7|8.5: what failed)>, e.g. feature/old (step 4: ls-remote exit 128, no delete planned) — or none
📍 Current branch: master (up to date with origin)
```

Every outcome goes on exactly one line, except rows marked `+`: a branch whose remote was deleted is always under Deleted branches, and its local copy may also appear under Left locally or Errors. Local copies are touched only as described in step 7 and 8.5.

| Outcome | Class | Summary line |
|---|---|---|
| PR not Ready in step 3, or excluded by the user | skip | Skipped: `#n (<state>)` / `<#n or branch> (excluded)` |
| Step 4: `ls-remote` returns nothing | — | not reported |
| Step 4: has commits after merge (`merge-base` exit 1) | kept | Kept |
| Step 4: `ls-remote` non-zero, or `merge-base` exit other than 0/1 | error (does not block step 8) | Errors (`step 4`) |
| 7.1: not ready, or head differs from `<headRefOid>` | skip | Skipped: `#n (not ready)` / `#n (head changed since plan)` |
| 7.2: merged | done | Merged (its branch is listed only if 7.3 ran) |
| 7.2: non-zero, head now differs from `<headRefOid>` | skip | Skipped: `#n (head changed since plan)` |
| 7.2: non-zero, head unchanged | error | Errors (`step 7`) |
| Pre-delete `ls-remote` or lease-delete re-check: returns nothing | skip | Already gone |
| … returns a different SHA | kept | Kept |
| … exits non-zero, or the re-check still returns `<expected sha>` | error | Errors (`step 7`) |
| Lease delete succeeds; no local copy, or `git branch -d` succeeds (step 7 or 8.5) | done | Deleted branches |
| `git branch -d` refuses: "not fully merged" (step 7 or 8.5) | skip | + Left locally: `<branch> (-d refused)` |
| `git branch -d` fails for another reason | error | + Errors (`step 7` / `step 8.5`) |
| Current branch, step 8 stopped before 8.5 | skip | + Left locally: `<branch> (current branch, not switched)` |
| Any other non-zero exit in step 7 (e.g. `gh pr view`) | error | Errors (`step 7`) |
| Step 8 | — | Current branch: `<main> (up to date with origin)`, `<main> (not fast-forwarded: <reason>)`, or `<branch> (not switched: uncommitted changes / git switch failed / errors in step 7)`; append `; fetch --prune failed` if the step-8 fetch failed |
