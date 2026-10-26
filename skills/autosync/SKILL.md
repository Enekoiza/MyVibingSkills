---
name: autosync
description: Groups pending changes into semantic commits and, if nothing is in doubt, pushes, opens a PR, runs the pr-auditor on it and fixes what it finds (up to 3 rounds) in one go. Stops and asks at any doubt.
argument-hint: [optional context: ticket, goal]
model: claude-sonnet-5-5
effort: medium
disable-model-invocation: true
---

# Autosync

Create semantic commits from all available changes, then push, open a PR,
audit it, and fix whatever the audit reports (critical findings, warnings and
suggestions, up to 3 rounds) — all in one run, without asking for
confirmation, **unless** a doubt comes up (see "Doubt gate"). At any doubt,
STOP and ask.

Do not add any --trailer.

Do not make one big commit by default.

Group files by purpose.

Commit each group separately.

$ARGUMENTS may contain extra context about the work done (ticket number, general
goal). If empty, infer it from the diff itself.

Talk to the user in English.

## Goal

Turn the current working tree into clean, meaningful commits — then land them
as an audited PR with its audit findings fixed.

No guessing. No mega commits. No vague messages. No mixing unrelated changes.

## Doubt gate

A doubt is anything where you would be guessing. When one appears, STOP right
there (before the next commit, push or PR), explain it in a few lines, propose
what you'd do, and wait for the user's answer. Do not continue with the rest
of the flow on your own.

Always treat these as doubts:

- You are on `main`, `master` or `develop`.
- No remote configured, or `gh` not installed or not authenticated.
- A file mixes unrelated changes that would need splitting by hunk.
- A change whose purpose you can't explain in one sentence, or that you can't
  assign confidently to a group.
- Anything that looks like a secret, a `.env` with real values, credentials,
  debug logs, temp files, editor files or build artifacts not already tracked.
- Changes that look like unrelated experiments or unfinished work
  (commented-out code, `console.log`/`Debug.WriteLine` left behind, `WIP`).
- A pre-commit hook fails.
- The push is rejected.
- The base branch for the PR is not obvious.
- A PR already exists for this branch (ask whether to just push and re-audit; if
  the user agrees, skip step 7 and continue from step 8 on that PR, fix loop
  included).
- The fix loop used its 3 iterations and the audit still reports critical
  findings, warnings or suggestions.
- The auditor raises a finding substantially the same as one from an earlier
  iteration.
- `fix-planner` returns **Blocked**, or `fix-implementer` returns
  `Status: FAILED` or `Status: PLAN_WRONG`.
- A fix iteration touches files well outside the findings it addresses, changes
  one of the files you deliberately left uncommitted, or its changes since the
  baseline don't match the implementer's **Changes** list.
- `fix-planner`, `fix-implementer` or `pr-auditor` is not available.

If none of these apply, run the whole flow straight through to the end of the
fix loop.

## Steps

### 1. Inspect repository state

```bash
git status --short
git diff --cached
git diff
git ls-files --others --exclude-standard
```

Understand every available change before committing. If there is nothing to
commit, say so and end.

### 2. Pre-flight checks

Check all of these before the first commit, so a problem is caught before any
history is written:

- Current branch (`git branch --show-current`) is not `main`, `master` or
  `develop`. If it is, stop and propose a branch name following
  `<type>/<kebab-case-slug>`.
- A remote is configured (`git remote -v`).
- `gh` is installed and authenticated (`gh auth status`).
- Base branch: the repo's default branch
  (`gh repo view --json defaultBranchRef -q .defaultBranchRef.name`), unless
  the branch clearly comes from another one (e.g. `develop`). If unclear, ask.
- Whether a PR already exists for the branch (`gh pr view --json url`).

Any failure is a doubt: stop and ask.

### 3. Detect issue key

Check the branch name and $ARGUMENTS for an issue key (e.g. `PROJ-123`,
`POW-456`, `#123`).

If there is a clear issue key, use it in every related commit. If not, commit
without it. Do not invent one.

### 4. Group changes semantically

Group files by intent. One commit = one purpose. Valid groups:

- one bug fix
- one feature
- one refactor
- one test update
- one documentation change
- one dependency update
- one config or CI change

If two files changed for the same reason, commit them together.

Stage files explicitly with `git add <file>`. Do not use `git add -A` when
changes are unrelated. Interactive staging (`git add -p`) is not available; if
a single file mixes unrelated changes, that's a doubt — stop and ask.

### 5. Create commits one by one

For each semantic group:

1. Stage only the files for that group.
2. Verify the staged diff (`git diff --cached`) and run the safety checks.
3. Create a Conventional Commit message.
4. Commit.
5. Check `git status --short` and repeat until no intentional changes remain.

Commit format:

```bash
git commit -F - <<'EOF'
<type>(<scope>): <summary>
EOF
```

With an issue key:

```bash
git commit -F - <<'EOF'
<issue-key>: <type>(<scope>): <summary>
EOF
```

The message goes through a quoted heredoc, so backticks and `$` are passed literally. Never put the message inside double quotes.

If a pre-commit hook fails, STOP, show the error, and ask how to proceed.
Never use `--no-verify` on your own.

### 6. Push

```bash
git push -u origin <current branch>
```

If the push is rejected, STOP and report it. Never `--force` or
`--force-with-lease` unless the user explicitly asks for it.

### 7. Open the PR

If a PR already exists for this branch and the user chose "just push and
re-audit", skip this step: keep that PR's number and URL and go to step 8.

```bash
gh pr create --base <base> --title '<title>' --body-file - <<'EOF'
<body>
EOF
```

- The body goes through a quoted heredoc, so backticks and `$` are passed literally. Never put the body inside double quotes.
- The title is single-quoted. Write any `'` in it as `'\''`.
- The `EOF` terminator must not appear on a line of its own in the body.

- Title: summarizes the whole batch of work, not a single commit.
- Body: what was done and why, the list of included commits, and how to verify
  it. Respect the repo's PR template (`.github/pull_request_template.md`) if
  one exists.
- Keep the PR URL for the final report.

### 8. Audit

- Run the `pr-auditor` subagent against the PR (the one just created, or the existing one from step 7),
  passing its number.
- Present the report it returns as-is.
- If `Result:` is `CLEAN`, or only 📦 dependency notes remain: skip step 9 and
  state the PR is ready to merge.
- Otherwise (any 🔴, 🟡 or 🔵 finding): go to step 9.
- Do NOT merge the PR yourself.

### 9. Fix loop (maximum 3 iterations)

Each iteration fixes the findings of the latest audit. 📦 Dependency notes are
only reported, never fixed.

1. **Dossier**: collect the findings of the latest audit verbatim (🔴, 🟡, 🔵),
   with file and line. From iteration 2 on, include only the new findings plus a
   short summary of what earlier iterations changed and why.
2. **Plan**: dispatch the `fix-planner` subagent with the dossier. Subagents
   start with no context, so paste everything it needs. Print the plan. If it
   returns **Blocked**, stop and ask.
3. **Baseline**: right before dispatching the implementer, record the working
   tree state: save the output of `git status --porcelain --untracked-files=all`,
   and run `git hash-object <path>` on every path it lists (note "absent" for
   deleted paths). This covers the files you deliberately left uncommitted.
   Take a fresh baseline every iteration, after the previous iteration's commit.
4. **Implement**: dispatch the `fix-implementer` subagent with the plan
   verbatim and the dossier, and tell it to implement the plan and nothing
   else. Read its status line: on `Status: FAILED` or `Status: PLAN_WRONG`, stop
   and ask. On `Status: DONE`, continue even if it lists pre-existing failures
   under **Noticed but not touched**; mention them in the final report.
5. **Scope check**: run `git status --porcelain --untracked-files=all` again
   and work out which paths changed since the baseline: paths that are new in
   the list, paths that left it, and baseline paths whose `git hash-object`
   differs. Ignore only build and test output (such as `bin/`, `obj/`,
   coverage files) that is untracked and git-ignored or clearly generated by
   the verification run. Stop and ask before committing if:
   - that set differs from the implementer's **Changes** list in either
     direction (a changed file it did not list, or a listed file that did not
     change);
   - any file you deliberately left uncommitted changed against its baseline;
   - any changed file is well outside the files named in the dossier and plan.
6. **Commit and push**: stage explicitly (`git add <file>`) only the paths in the
   implementer's **Changes** list (which the scope check confirmed match the
   changes since the baseline), never a file you deliberately left
   uncommitted. Verify the staged diff (`git diff --cached`) and run the safety
   checks, exactly as in step 5. Commit using the commit format from step 5 (same
   issue key and message rules), choosing the type from "Commit types" according
   to what the fix addresses (`fix` for bugs, `docs`, `style`, `refactor`,
   `test`, etc.); when one iteration mixes types, use the type of its most
   significant change. Then `git push`. Never force-push.
7. **Re-audit**: run `pr-auditor` on the PR again and present the report.

After the re-audit:

- `CLEAN`, or only 📦 notes remain: the loop ends, the PR is ready to merge.
- Findings remain and fewer than 3 iterations ran: start the next iteration.
- Findings remain after the 3rd iteration: STOP. Keep the branch and PR as they
  are, list what is still open, and ask the user how to proceed.

Guards (each one is a STOP-and-ask):

- A finding is substantially the same as one from an earlier iteration. Likely a
  false positive or a disagreement; describe both sides.

Never weaken, skip or delete a test to make the suite green. Do not expand a fix
beyond its finding.

## Commit types

- `feat`: new feature
- `fix`: bug fix
- `docs`: documentation only
- `style`: formatting only, no logic change
- `refactor`: code change without behavior change
- `perf`: performance improvement
- `test`: tests added or updated
- `build`: build system or dependencies
- `ci`: CI/CD changes
- `chore`: maintenance
- `revert`: revert previous commit

## Scope

Use a short scope when useful: feature area, package, route, module, service
or config name. Skip it only if it adds no value (`chore: Update dependencies`).

```bash
fix(login): Show invalid credentials error
feat(dashboard): Add revenue chart
test(api): Cover pagination params
ci(github): Cache pnpm dependencies
```

## Message rules

- Max 72 characters.
- Imperative mood: `Add`, `Fix`, `Update`, `Remove`.
- Capitalize the summary. No period at the end.
- Be specific. Describe the purpose, not just the file changed.
- Do not mention implementation details unless they are the point.
- Never `changes`, `stuff`, `misc`, `updates` or `wip`.

## Splitting rules

- UI change + dependency update = two commits
- bug fix + test for that bug = usually one commit
- refactor + behavior change = two commits
- docs for a feature + feature code = usually one commit
- formatting many files + logic change = two commits
- generated lockfile from dependency update = same commit as dependency update

If a change cannot be explained by the same sentence, split it.

## Safety rules

Never commit secrets, `.env` files with real values, API keys, tokens,
credentials, debug logs, local editor files, temporary files, build artifacts
(unless intentionally tracked) or unrelated experiments. Finding any of them
is a doubt: leave it unstaged and ask.

If the staged diff contains unrelated changes, unstage and regroup:

```bash
git restore --staged <file>
```

## Final report

Run `git status --short`, then report in as few words as possible:

- commits created
- push and PR status (URL)
- audit result: fix iterations run and what each fixed, then ready to merge or
  the findings still open
- files intentionally left uncommitted
- anything skipped for safety

If the run stopped at a doubt, report what was already done (commits created,
pushed or not) and what is pending the user's answer.

Done means clean semantic history landed as an audited PR with a clean audit
(or the open findings handed to the user), not just zero pending files.
