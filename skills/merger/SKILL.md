---
name: merger
description: Merges the open pull requests of the current repository that are ready (no conflicts, green checks, not draft) and deletes branches whose code is already merged into the main branch (master/main), both on the remote and locally; local branches with unmerged changes are deleted only if the user names them. Invoke manually with /merger.
argument-hint: "[merge|squash|rebase]"
model: claude-sonnet-5-5
effort: medium
disable-model-invocation: true
allowed-tools: Bash(gh auth status:*), Bash(gh repo view:*), Bash(gh pr list:*), Bash(gh pr view:*), Bash(git fetch:*), Bash(git branch --show-current*), Bash(git merge-base --is-ancestor *), Bash(git status:*), Bash(git rev-parse:*), Bash(git ls-remote:*), Bash(git pull --ff-only:*), Bash(git for-each-ref:*), Bash(git worktree list:*), Bash(git rev-list --count *)
---

# Merger

Merges the open PRs that are ready and cleans up branches, remote and local, whose code is already in the main branch. Always works on the repository in the current directory and uses `gh` (GitHub CLI).

Always talk to the user in English.

Merging and deleting branches are hard to undo, so this skill first analyses, then shows a plan, and only acts once the user confirms.

**References.** "Section N", "step N" and "N." all mean numbered heading N below. "1.5", "4b.3", "7.2" mean item 5 of section 1, item 3 of section 4b, item 2 of section 7; "4b.3.2" and "8.5.3" are sub-items of those items; "4 (b)" and "4b.4 (b)" are lettered bullets.

**Shell safety.** Branch names, ref names and SHAs come from git and GitHub (including fork PRs and branches created by `gh pr checkout`) and may contain `$`, `;`, `&`, `|`, `<`, `>`, `(`, `)`, backticks or quotes. They are data, never shell syntax:
- Put every branch name, ref and SHA into a command inside single quotes, exactly as returned by git or `gh`; write a `'` inside it as `'\''`. Never use double quotes and never leave it unquoted.
- In `git branch -D`, put `--` before the branch name. Wherever a command below uses a full ref (`refs/heads/…`, `refs/remotes/origin/…`), keep it: a short name can be read as an option or resolve to another ref (e.g. a local branch called `origin/main`).
- In the snippets below, a placeholder inside quotes stands for the value with any `'` already written as `'\''`.

## 1. Pre-flight checks

1. `gh auth status` — if there is no session, stop and tell the user to run `gh auth login`.
2. `git rev-parse --is-inside-work-tree` — if it is not a git repo, stop.
3. `git fetch --prune origin` to bring the remote state up to date.
4. Detect the main branch:
   `gh repo view --json defaultBranchRef,mergeCommitAllowed,squashMergeAllowed,rebaseMergeAllowed`
   It will usually be `master`. From here on it is called the **main branch**.
5. `git branch --show-current` — record the output as the **starting branch** (empty output = detached HEAD: no starting branch).

If any command in sections 1–5 exits non-zero and section 9 gives it no other outcome, stop and tell the user what failed; nothing has been changed yet.

## 2. Merge method

- If the user passed an argument (`$ARGUMENTS` = `merge`, `squash` or `rebase`), use that.
- Otherwise, use the first one the repo allows in this order: `merge`, `squash`, `rebase`.
- If the requested method is not allowed in the repo, say so and use an allowed one.
- If $ARGUMENTS is not empty and is not exactly one of merge, squash or rebase (e.g. fast, squash please), say it is not a merge method and ask which one to use before continuing; nothing has been changed yet. If the user does not pick one, stop.

## 3. Analyse the open PRs

```bash
gh pr list --state open --limit 1000 \
  --json number,title,author,headRefName,headRefOid,baseRefName,isDraft,mergeable,mergeStateStatus,reviewDecision,statusCheckRollup,headRepositoryOwner,isCrossRepository,createdAt
```

Record each PR's `headRefOid`: it is the head SHA shown in the plan and the only commit that may be merged.

Classify each PR:

| State | Condition | Action |
|---|---|---|
| ✅ Ready | not draft, `mergeable = MERGEABLE`, `mergeStateStatus` is `CLEAN` or `HAS_HOOKS`, and no Skip row below matches (❔ Not ready aside) | Merge |
| ⏳ Pending checks | any check in `PENDING`/`IN_PROGRESS`/`QUEUED` | Skip |
| ❌ Failing checks | any check in `FAILURE`/`ERROR`/`CANCELLED`/`TIMED_OUT` | Skip |
| ⚠️ Conflicts | `mergeable = CONFLICTING` or `mergeStateStatus = DIRTY` | Skip |
| 🔄 Out of date | `mergeStateStatus = BEHIND` | Skip (say it needs updating with the base branch) |
| 🔒 Blocked | `mergeStateStatus = BLOCKED` or `reviewDecision = CHANGES_REQUESTED` / `REVIEW_REQUIRED` | Skip |
| 📝 Draft | `isDraft = true` | Skip |
| ❓ Unknown | `mergeable = UNKNOWN` or `mergeStateStatus = UNKNOWN` | Query once more with `gh pr view <n> --json mergeable,mergeStateStatus` (GitHub computes it in the background) and classify the new values with this table; if either is still `UNKNOWN`, skip |
| ❔ Not ready | no row above matches (e.g. `mergeStateStatus = UNSTABLE` with checks in states not listed above, such as `ACTION_REQUIRED` or `STARTUP_FAILURE`) | Skip, shown as `not ready (<mergeStateStatus>)` |

When several Skip rows match, show the first one listed.

Never force a merge with `--admin` or by bypassing branch protections.

## 4. Find already-merged branches (no open PR)

Besides the open PRs, find remote branches whose code is already in the main branch but which still exist:

```bash
gh pr list --state merged --base '<main branch>' --limit 1000 --json number,headRefName,headRefOid,isCrossRepository,mergedAt
```

Group the merged PRs by `headRefName` (one branch can have several), ignoring PRs with `isCrossRepository = true` and branches protected by section 5, and run this check per candidate branch. This also covers squash merges, which `git branch --merged` does not detect.

- (a) Run `git ls-remote origin 'refs/heads/<branch>'`. Everywhere in this skill, "returns nothing" means it exited 0 with empty output: the branch is already gone. A non-zero exit is always an error, never 'already gone'. Here, a branch that returns nothing is dropped and not reported; a non-zero exit is a step-4 error: plan no delete for that branch.
- (b) Take the full 40-character SHA it returns as the **current tip**.
- (c) The branch is deletable only if one of these holds, checked in this order:
  - first, the tip equals the `headRefOid` of any merged PR from that branch (this covers squash and rebase merges); if so, the branch is deletable and `merge-base` is not run;
  - only otherwise, `git merge-base --is-ancestor '<tip>' 'refs/remotes/origin/<main branch>'` exits 0. Exit code 1 means "not an ancestor"; it is not an error. Any other exit code (e.g. 128 when the tip object is missing locally) is an error, never 'kept': do not plan a delete for that branch and report it under errors.
- (d) Otherwise, if neither condition holds and nothing in (a) or (c) was an error, classify the branch as **"has commits after merge — kept"**. Never delete it.

## 4b. Classify local branches

Besides the remote branches, look at the local copies, including branches that were never pushed or whose remote is already gone.

1. Run both:
   - `git for-each-ref refs/heads --format='%(refname:lstrip=2)%09%(objectname)%09%(upstream:short)%09%(upstream:track)%09%(upstream:remoteref)'` — the second column is the full 40-character SHA, called the **local tip**. An empty third column means the branch has no upstream (it may never have been pushed, or was pushed without `-u`); `[gone]` in the fourth column means the upstream no longer exists. The fifth column is the upstream's branch on its remote, as `refs/heads/<name>` (e.g. `refs/heads/develop` for a branch tracking `origin/develop`); it is empty when there is no upstream.
   - `git rev-parse --verify --quiet 'refs/remotes/origin/<main branch>'` — its output is the **main tip**. Every comparison with the main branch in 4b uses this SHA, never the short name `origin/<main branch>`.
2. `git worktree list --porcelain` — a branch checked out in a worktree other than the current one, and not excluded by 4b.3, is excluded and reported under Skipped as `<branch> (checked out in another worktree)`.
3. Exclude a branch at the first of these that matches. An excluded branch is never deleted and cannot be named in section 6.
   1. **Protected by its own name** (not reported): the main branch; `master`, `main`, `develop`, `dev` and any `release/*` or `hotfix/*` name, whether or not the repo uses it; the base of any open PR from section 3; the head of any open PR from section 3, except the head of a PR with `isCrossRepository = false` that the plan merges into the main branch, which stays a candidate for `after PR #n merges`. That exception never applies to a branch that is also the base of an open PR, the head of a fork PR, or the head of an open PR the plan does not merge into the main branch.
   2. **Protected through its upstream**: if the fifth column is `refs/heads/<name>`, whatever the remote (including `.` for a local upstream), apply 4b.3.1 to `<name>` with the same exceptions. If `<name>` is protected, report the branch under Skipped as `<branch> (tracks protected <name>)` (e.g. `local-dev` tracking `origin/develop`, or `feat` created with `git switch -c feat origin/<main branch>`).
   3. **Starting branch with no commits beyond the main branch**: the starting branch (1.5) when its local tip equals the local tip of the main branch (4b.1, if that branch exists) or `git merge-base --is-ancestor '<local tip>' '<main tip>'` exits 0 (its tip is the main tip or an older commit of it, e.g. a branch just created from an older main). Report it under Skipped as `<branch> (starting branch, no commits beyond <main>)`. If that `merge-base` exits other than 0 or 1, exclude the branch and report it under Errors (`step 4b`). Otherwise the starting branch is classified normally, and 8.5.1 skips it only if it is still checked out.
4. Classify each remaining branch:
   - **Merged**, if one of these holds, checked in this order (plan label in brackets):
     - (a) the local tip equals the `headRefOid` of any merged PR with `isCrossRepository = false` from the step-4 query (this covers squash and rebase merges) [`merged: tip is PR #n's head`];
     - (b) the local tip equals the `headRefOid` recorded in section 3 of a PR with `isCrossRepository = false` that the plan will merge into the main branch [always shown as `after PR #n merges`, see 4b.5];
     - (c) `git merge-base --is-ancestor '<local tip>' '<main tip>'` exits 0 [`merged: ancestor of <main>`, or `merged: no commits beyond <main>` when the local tip equals the main tip; such a branch is deleted too]. Exit code 1 means "not an ancestor", not an error. Any other exit code is an error: plan no delete for that branch and report it under Errors (`step 4b`).
   - **Unmerged** otherwise. Record `git rev-list --count '<main tip>..<local tip>'` (commits not in the main branch; if it fails, show `unknown number of commits`, which is not an error) and the upstream status (tracking, `[gone]` or no upstream) so the plan can show them.
5. For every branch classed Merged, record as its **tied PRs** every PR with `isCrossRepository = false` that the plan will merge into the main branch and whose `headRefOid` recorded in section 3 equals the local tip, whatever rule classed it and whatever its name. A branch with tied PRs is shown as `after PR #n merges` (listing all of them, e.g. `after PRs #12, #13 merge`) instead of its 4b.4 label. Step 8.5 deletes it only if 7.2 merged every one of them. Unmerged branches never have tied PRs: a tip equal to a planned head always matches 4b.4 (b).

A failure never stops the run here. If any command in 4b.1 or 4b.2 fails (including the `rev-parse` when `origin/<main branch>` does not exist), skip all local cleanup and report it under Errors (`step 4b`). A failure on a single branch (`merge-base` exit other than 0/1) affects only that branch.

Local branches are deleted only in step 8.5, always with the SHA recorded here.

## 5. Branches that are NEVER deleted

These rules apply to remote and local branches alike. For local branches, 4b.3 spells out how they apply (the exception for heads of PRs the plan merges, the upstream check, and the starting branch).

- The main branch and `master`, `main`, `develop`, `dev`, `release/*`, `hotfix/*` when they are long-lived branches of the repo.
- Any branch that is the **base** of another open PR (deleting it would close that PR).
- Any branch that is the **head** of another open PR.
- Fork branches (`isCrossRepository = true`): they are not ours, do not touch them.

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

Local branches to delete (L):
  feature/login   @c3d4e5f  (merged: tip is PR #9's head)
  feature/export  @a1b2c3d  (after PR #12 merges)
  docs/old        @0a1b2c3  (merged: ancestor of master)

Local — unmerged changes, kept unless named as <branch> (force-delete) (U):
  wip/search      @7c8d9e0  (3 commits not in master, no upstream)
  fix/typo        @1d2e3f4  (1 commit not in master, upstream gone)

Errors (E):
  feature/old  ls-remote failed (exit 128) — no delete planned

Skipped (K):
  #14  ❌ failing checks (build)
  #16  ⚠️ conflicts with master
```

In To be merged, show `(branch will be deleted)` only when 7.3 would delete the branch as planned: the PR's base is the main branch, it is not a fork PR, its branch is not protected by section 5 (e.g. it is also the head or base of another open PR) and the user has not excluded its branch delete. Otherwise show `(branch is kept)`.

Then ask: "Shall I proceed with everything, exclude any (name them), force-delete any of the unmerged local branches (write `<branch> (force-delete)`), or cancel?"

1. **Confirmation.** Do not continue without an explicit yes to the plan exactly as last shown. A reply that excludes items or names branches is never that yes. Never include an unmerged branch on your own initiative.
2. **Nothing to do.** If there is nothing to merge or delete (no PR, no remote branch, no local branch to delete), show the plan without the question and finish; unmerged local branches alone do not count as something to delete.
3. **Cancel.** If the user cancels, or excludes every merge and delete, stop: no step 7, step 8 or summary.
4. **Exclusions.** Excluded items are not touched. Only `#n` excludes a merge; a branch name never does. The user can exclude:
   - a PR, as `#n`: it is not merged and its branch is kept (local effects in 6.6);
   - only the branch delete of a PR to be merged, by naming its head branch bare or as `<branch> (remote)`: its remote branch is kept, and so is every local entry whose name is that PR's `headRefName` or one of whose tied PRs it is (each shown as `<branch> (local, excluded)`); the PR stays under To be merged with `(branch is kept)`;
   - an already-merged branch, as `<branch>` or `<branch> (remote)`;
   - a local branch to delete, as `<branch>` or `<branch> (local)`.
   A bare name excludes every planned delete with that name, remote and local (e.g. a name listed under both Already-merged branches to delete and Local branches to delete excludes both copies); it never force-deletes an unmerged local branch with that name (6.5). For a name listed under both Already-merged branches to delete and Local branches to delete, `<branch> (remote)` or `<branch> (local)` excludes only that copy; for a PR's head branch, `<branch> (remote)` also keeps its local entries, as in the second bullet above.
5. **Naming unmerged branches.** Unmerged local branches are kept unless the user names them, and the only way to name one is `<branch> (force-delete)`, with the name exactly as listed under Local — unmerged changes. A bare name, `<branch> (remote)` or `<branch> (local)` never names a branch, even when an unmerged local branch has that name: if it matches a planned delete, 6.4 applies and the unmerged local branch stays kept; if it matches no planned delete but is listed under Local — unmerged changes, change nothing, say that force-deleting needs `<branch> (force-delete)`, and ask the same question again. A named branch moves into Local branches to delete marked `(unmerged, force-delete)`; its commits are lost except through the reflog. If the user writes `(force-delete)` after any other branch, do not add it: say why (protected by section 5, by name or through its upstream; checked out in another worktree; a step-4b error; kept as the starting branch with no commits beyond <main>; already in Local branches to delete; or not a local branch) and ask the same question again.
6. **Excluding a PR, local side.** Drop every local entry one of whose tied PRs is the excluded PR, then re-apply 4b.3 to both local lists (to delete and unmerged). The excluded PR's head is now the head of an open PR the plan does not merge, so 4b.3 protects a local branch with that name, whatever its 4b class. Dropped entries with that name are not reported; every other dropped entry is shown under Skipped as `<branch> (local, excluded)`.
7. **Ask again.** After any exclusion or naming (unless 6.3 applies), show the adjusted plan, with excluded items under Skipped as `#n (excluded)`, `<branch> (remote, excluded)` or `<branch> (local, excluded)`, and ask the same question again.

## 7. Execute

`<headRefOid>` is the head SHA recorded for the PR in step 3; `<plan sha>` is the full 40-character current tip recorded for the branch in step 4 (b). The plan shows both shortened, for display only. Classify every outcome with the table in section 9; only an **error** in this step stops step 8 from switching branch. Never retry a failed command.

Merge the PRs in creation order (oldest first). For each PR:

1. Query its state again, because merging one PR can cause conflicts or leave the following ones `BEHIND`: `gh pr view <n> --json isDraft,mergeable,mergeStateStatus,reviewDecision,statusCheckRollup,headRefOid,baseRefName`; if `mergeable` or `mergeStateStatus` is `UNKNOWN` (common right after the previous merge), run the same query once more. Skip the PR if its `headRefOid` differs from `<headRefOid>`, its `baseRefName` differs from the base in the plan, or it is no longer ready (ready = the step-3 table classifies the latest values as ✅ Ready; do not run that table's Unknown re-query again, and a value still `UNKNOWN` is not ready). Never merge a head, or into a base, the user did not see.
2. `gh pr merge <n> --<method> --match-head-commit '<headRefOid>'`. Never pass `--delete-branch` (gh deletes the local branch with `git branch -D`). Then, whatever the exit code, run `gh pr view <n> --json state,headRefOid,baseRefName` once and classify the result with section 9, by `state` first. Exit 0 alone is not a merge: on a base with a merge queue, gh adds the PR to the queue or enables auto-merge and still exits 0. Only exit 0 with `state = MERGED` is merged; this is the only meaning of "7.2 merged" in this skill. In every other case, including `state = MERGED` with a non-zero exit, do not touch its branch, remote or local.
3. Only if 7.2 merged the PR, the `baseRefName` from 7.2 equals the base in the plan and that base is the main branch, and its branch is neither protected by section 5 nor excluded: run `git ls-remote origin 'refs/heads/<branch>'`; only if it returns `<headRefOid>`, **lease delete** the branch with `<expected sha>` = `<headRefOid>`. PRs to another base (including a base changed since the plan) and protected branches: leave the branch alone.

Then, for each already-merged branch from step 4, immediately before deleting it run `git ls-remote origin 'refs/heads/<branch>'`; only if it returns `<plan sha>`, **lease delete** it with `<expected sha>` = `<plan sha>`.

**Lease delete.** `git push --force-with-lease='refs/heads/<branch>:<expected sha>' origin --delete 'refs/heads/<branch>'`. Never add `--force`. If it exits non-zero for any reason (stale-info rejection, `remote ref does not exist`, or anything else), the remote delete did not happen: run `git ls-remote origin 'refs/heads/<branch>'` exactly once and classify the result with section 9.

**Local delete.** Never delete local branches in step 7. All local deletes happen in step 8.5, from the list planned in step 4b; this step only records which PRs 7.2 merged (as defined in 7.2), because 8.5.2 depends on it.

## 8. Leave the repo on the main branch

If there was any error in step 7, do not switch branch: run `git fetch --prune origin`, then go to 8.5 and leave the user where they were so they can review. Otherwise (step-4 errors, skips and kept branches do not count):

1. `git status --porcelain` — if it fails or shows uncommitted changes, **do not switch branch**; go to 8.5.
2. `git switch '<main branch>'` — if it fails, go to 8.5.
3. `git pull --ff-only origin '<main branch>'` — if it fails (e.g. the local main branch has commits that are not on origin), force nothing, leave the branch as it is and continue.
4. `git fetch --prune origin` — if it fails, continue with 8.5.
5. **Local cleanup.** Run it whether or not the branch switch happened. For each branch in the plan's Local branches to delete list (merged ones, plus unmerged ones the user named), except those the user excluded:
   1. `git branch --show-current` — if it is this branch, skip it (Left locally: checked out). If the command fails, do not delete any local branch: report it once under Errors (`step 8.5`) and end the local cleanup.
   2. If the branch's name is the `headRefName` of a PR the plan merges, or 4b recorded tied PRs for it, whatever its 4b class (e.g. an `after PR #n merges` branch, one classed Merged by another rule, or one the user named whose tip differs from that PR's head), skip it unless 7.2 merged every such PR (the one whose `headRefName` it is and all its tied PRs) into the planned base and that base is the main branch (label it after the first such PR, in creation order, that fails this: Left locally: `<branch> (PR not merged)`, `<branch> (PR merge error)` when 7.2 ended in the `gh exited <N> but PR is merged` error, or `<branch> (PR merged into another base)`).
   3. `git rev-parse --verify --quiet 'refs/heads/<branch>'` — it must print `<local tip>`, the full SHA recorded in step 4b. No output (exit 1) means the branch is already gone (Already gone); a different SHA means it changed since the plan: keep it (Kept: changed since plan). Any other exit code is an error: do not delete.
   4. `git branch -D -- '<branch>'`. `-D` is intended here because squash and rebase merges are never ancestors of the main branch; the confirmed plan and the SHA check in 8.5.3 are the safety guard. Never retry a failure.

## 9. Final summary

```
✅ Merged: #12, #15
🗑️  Deleted branches (remote): feature/export, feature/login
💻 Deleted locally: feature/export, feature/login, docs/old, wip/search (force-deleted)
🛡️  Kept (commits after merge / changed since plan): fix/old-paging (remote, commits after merge)
ℹ️  Already gone: <branch (remote) | branch (local) — or none>
📌 Left locally: <branch (checked out | PR not merged | PR merge error | PR merged into another base) | unmerged: wip/other — or none>
⏭️  Skipped: #14 (failing checks), #16 (conflicts), #17 (head changed since plan)
⚠️  Errors: <#n or branch (step 4|4b|7|8.5: what failed)>, e.g. feature/old (step 4: ls-remote exit 128, no delete planned) — or none
📍 Current branch: master (up to date with origin)
```

Every outcome goes on exactly one line, except rows marked `+`: a PR whose branch delete was excluded also has `<branch> (remote, excluded)` under Skipped, but only if the PR itself is under Merged; otherwise that remote line is left out. This applies only to the remote line: the PR's excluded local entries are always reported as `<branch> (local, excluded)`. Remote and local copies of the same branch are separate outcomes, each reported on its own line: the remote one under Deleted branches, Kept, Already gone (`<branch> (remote)`), Skipped (`<branch> (remote, excluded)`) or Errors; the local one under Deleted locally, Left locally, Kept, Already gone (`<branch> (local)`), Skipped (`<branch> (local, excluded)`, `<branch> (checked out in another worktree)`, `<branch> (tracks protected <name>)` or `<branch> (starting branch, no commits beyond <main>)`) or Errors. Local copies are deleted only in step 8.5. Unmerged local branches the user did not name are listed under Left locally as `unmerged: <branch>`.

| Outcome | Class | Summary line |
|---|---|---|
| Non-zero exit in sections 1–5 with no other row (e.g. `gh pr list`, the step-3 Unknown re-query, `git branch --show-current`) | stop | no summary: say what failed; nothing was changed |
| User cancels, or excludes every merge and delete | stop | no summary: say nothing was changed |
| PR not Ready in step 3, or excluded by the user | skip | Skipped: `#n (<state>)` (the step-3 row name, e.g. `failing checks (build)`, `not ready (UNSTABLE)`) / `#n (excluded)` or `<branch> (remote, excluded)` (`+` when only a PR's branch delete is excluded; see above) |
| Step 4: `ls-remote` returns nothing | — | not reported |
| Step 4: has commits after merge (`merge-base` exit 1) | kept | Kept: `<branch> (remote, commits after merge)` |
| Step 4: `ls-remote` non-zero, or `merge-base` exit other than 0/1 | error (does not block step 8) | Errors (`step 4`) |
| 7.1: head differs, else base differs, else not ready (first that applies) | skip | Skipped: `#n (head changed since plan)` / `#n (base changed since plan)` / `#n (not ready: <state>)` |
| 7.2: `state = MERGED`, `gh pr merge` exited non-zero | error | Errors: `#n (step 7: gh exited <N> but PR is merged into <base>; remote branch kept)`; its local copy, if planned for deletion, is reported separately by 8.5.2 as `<branch> (PR merge error)` |
| 7.2: `state = MERGED`, exit 0, `baseRefName` differs from the plan | done | Merged: `#n (into <base>, base changed since plan; branch kept)` |
| 7.2: `state = MERGED`, exit 0, `baseRefName` as in the plan | done | Merged: `#n` (its branch appears only through the 7.3 rows if 7.3 ran, or as `+` `<branch> (remote, excluded)`) |
| 7.2: `state = CLOSED` | skip | Skipped: `#n (closed without merge, branch kept)` |
| 7.2: `state = OPEN`, exit 0 (queued or auto-merge enabled) | skip | Skipped: `#n (queued / auto-merge enabled, branch kept)` |
| 7.2: `state = OPEN`, non-zero, head now differs from `<headRefOid>` | skip | Skipped: `#n (head changed since plan)` |
| 7.2: `state = OPEN`, non-zero, head unchanged | error | Errors (`step 7`) |
| Pre-delete `ls-remote` or lease-delete re-check: returns nothing | skip | Already gone (`<branch> (remote)`) |
| … returns a different SHA | kept | Kept: `<branch> (remote, changed since plan)` |
| … exits non-zero, or the re-check still returns `<expected sha>` | error | Errors (`step 7`) |
| Lease delete succeeds | done | Deleted branches |
| Step 4b: `merge-base` exit other than 0/1 (4b.3.3 or 4b.4 (c)) | error (does not block step 8) | Errors (`step 4b`) |
| Step 4b: `git rev-list --count` fails | — | not reported; the plan shows `unknown number of commits` |
| Step 4b: `git for-each-ref`, the main-tip `git rev-parse` or `git worktree list` fails | error (no local cleanup; does not block step 8) | Errors (`step 4b`) |
| Step 4b: checked out in another worktree | skip | Skipped: `<branch> (checked out in another worktree)` |
| Step 4b: excluded through its upstream (4b.3.2) | skip | Skipped: `<branch> (tracks protected <name>)` |
| Step 4b: starting branch with no commits beyond the main branch (4b.3.3) | skip | Skipped: `<branch> (starting branch, no commits beyond <main>)` |
| Step 4b: unmerged and not named by the user | skip | Left locally: `unmerged: <branch>` |
| Step 6: local delete excluded by the user directly, through a branch-delete-only exclusion of its PR, or dropped by 6.6 under a name other than the excluded PR's head (entries with that head's name are not reported) | skip | Skipped: `<branch> (local, excluded)` |
| 8.5: `git branch -D` succeeds (merged, or unmerged and named) | done | Deleted locally (`(force-deleted)` if it was unmerged) |
| 8.5.1: branch is checked out | skip | Left locally: `<branch> (checked out)` |
| 8.5.2: its PR was not merged in 7.2 | skip | Left locally: `<branch> (PR not merged)`, or `<branch> (PR merge error)` after the 7.2 `gh exited <N> but PR is merged` error |
| 8.5.2: its PR was merged into a base other than the planned main branch | skip | Left locally: `<branch> (PR merged into another base)` |
| 8.5.1: `git branch --show-current` fails | error (ends local cleanup) | Errors (`step 8.5`) |
| 8.5.3: `rev-parse` prints nothing (exit 1) | skip | Already gone (`<branch> (local)`) |
| 8.5.3: `rev-parse` prints a different SHA | kept | Kept: `<branch> (local, changed since plan)` |
| 8.5.3: `rev-parse` exits other than 0/1, or 8.5.4 `git branch -D` fails | error | Errors (`step 8.5`) |
| Any other non-zero exit in step 7 not covered above (e.g. a `gh pr view`) | error | Errors (`step 7`) |
| Step 8 (branch switch) | — | Current branch: `<main> (up to date with origin)`, `<main> (not fast-forwarded: <reason>)`, or `<starting branch, or detached HEAD> (not switched: uncommitted changes / git status failed / git switch failed / errors in step 7)`; append `; fetch --prune failed` if the step-8 fetch failed |
