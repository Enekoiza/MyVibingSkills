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

Group the merged PRs by `headRefName` (one branch can have several) and run this check per candidate branch. This also covers squash merges, which `git branch --merged` does not detect.

- (a) Run `git ls-remote --heads origin <branch>`. If it returns nothing, the branch is already gone: skip it.
- (b) Take the SHA it returns as the **current tip**.
- (c) The branch is deletable only if one of these holds:
  - the tip equals the `headRefOid` of any merged PR from that branch (this covers squash and rebase merges);
  - `git merge-base --is-ancestor <tip> origin/<main branch>` exits 0. Exit code 1 means "not an ancestor"; it is not an error.
- (d) Otherwise, classify the branch as **"has commits after merge — kept"**. Never delete it.

As a supplement, `git branch -r --merged origin/<main branch>` lists the branches merged with a regular merge commit.

## 5. Branches that are NEVER deleted

- The main branch and `master`, `main`, `develop`, `dev`, `release/*`, `hotfix/*` when they are long-lived branches of the repo.
- Any branch that is the **base** of another open PR (deleting it would close that PR).
- Any branch that is the **head** of another open PR.
- Fork branches (`isCrossRepository = true`): they are not ours, do not touch them.
- The local branch the user is currently on (`git branch --show-current`); in that case delete only the remote one and mention it.

Only delete branches of PRs whose base is the **main branch**. If a PR targets another branch (for example, a feature going into `develop`), it is merged but its branch is kept, because its code is not yet in the main branch.

## 6. Show the plan and ask for confirmation

Show a compact summary before doing anything:

```
Repo: <owner>/<repo> · main branch: master · method: merge

To be merged (N):
  #12  Add export endpoint        feature/export @a1b2c3d → master  (branch will be deleted)
  #15  Fix pagination             fix/paging @b2c3d4e     → develop (branch is kept)

Already-merged branches to delete (M):
  feature/login  @a1b2c3d  (PR #9, merged on 2026-09-10)

Kept — commits after merge (J):
  fix/paging  @d4e5f6a  (PR #7 merged @9f8e7d6; tip differs)

Skipped (K):
  #14  ❌ failing checks (build)
  #16  ⚠️ conflicts with master
```

Ask: "Shall I proceed with everything, exclude any, or cancel?". Do not continue without an explicit yes. If there is nothing to do, say so and finish.

## 7. Execute

Merge the PRs in creation order (oldest first). **Before each merge**, query its state again, because merging one PR can cause conflicts or leave the following ones `BEHIND`:

```bash
gh pr view <n> --json mergeable,mergeStateStatus,headRefOid
```

If it is no longer ready, or its `headRefOid` differs from the one recorded in step 3, skip it. Note it for the summary as 'not ready' or 'head changed since the plan — not merged'. Never merge a head the user did not see.

- PR against the main branch and a deletable branch:
  - `gh pr merge <n> --<method> --match-head-commit <headRefOid>`.
  - Then run `git ls-remote --heads origin <branch>`. If it returns nothing, the branch is already gone; skip the delete. If the SHA differs from `<headRefOid>`, do not delete it: record it as 'commits pushed after merge — kept' in the summary.
  - Only if it equals `<headRefOid>`, run `git push origin --delete <branch>`.
  - Never pass `--delete-branch`: gh deletes the local branch with `git branch -D`.
  - Delete the local branch with `git branch -d <branch>` only if the remote delete happened; if it is the current branch, do it in step 8.5.
- PR against another branch, or a branch protected by section 5:
  `gh pr merge <n> --<method> --match-head-commit <headRefOid>`

`<headRefOid>` is the value recorded in step 3. If `gh pr merge` exits non-zero (including a head-commit mismatch), the PR was not merged. Record it as skipped with the error, do not retry, and do not delete its branch.

For the already-merged branches from step 4, immediately before deleting each one run `git ls-remote --heads origin <branch>` again. If the SHA differs from the one shown in the plan, do not delete it: record the branch as "changed since the plan, kept" in the summary.

```bash
git push origin --delete <branch>
git branch -d <branch>      # only if it exists locally; use -d, never -D
```

If `git branch -d` fails because git does not recognise the branch as merged (typical after a squash), do not use `-D` automatically: leave the local branch and mention it in the summary.

If a command fails, do not retry blindly: record the error and continue with the next PR.

## 8. Leave the repo on the main branch

If everything went well (no errors in step 7), leave the local repo on the main branch and up to date:

1. `git status --porcelain` — if there are uncommitted changes, **do not switch branch**: note that it stayed on `<current branch>` because of pending changes and move on to the summary.
2. `git switch <main branch>`
3. `git pull --ff-only origin <main branch>` — if it cannot fast-forward (the local master has commits that are not on origin), force nothing: note it and leave the branch as it is.
4. `git fetch --prune origin`
5. If in step 7 any local branch could not be deleted because it was the current branch, delete it now with `git branch -d <branch>` (again, never `-D`).

If there was any error in step 7, do not switch branch: only run `git fetch --prune origin` and leave the user where they were so they can review.

## 9. Final summary

```
✅ Merged: #12, #15
🗑️  Deleted branches: feature/export, feature/login
🛡️  Kept (commits after merge): <branches or none>
⏭️  Skipped: #14 (failing checks), #16 (conflicts), #17 (head changed since plan)
⚠️  Errors: none
📍 Current branch: master (up to date with origin)
```
