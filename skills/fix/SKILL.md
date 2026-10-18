---
name: fix
description: Plans a fix with Opus 5.5 (medium effort), implements it with Sonnet 5.5 (medium effort), pushes, opens a PR if there is none, then loops through pr-auditor until the audit reports no critical findings and no warnings.
argument-hint: [what to fix — leave empty to use the previous message]
allowed-tools: Read, Grep, Glob, Bash, Agent, TodoWrite
model: claude-sonnet-5-5
effort: medium
disable-model-invocation: true
---

# Fix

Closed loop: plan → implement → push → PR → audit → repeat until `pr-auditor` reports 0 🔴 critical and 0 🟡 warnings.

You are the **orchestrator**. You do not write production code yourself and you do not plan the
fix yourself: the `fix-planner` subagent plans (Opus 5.5, medium effort) and the `fix-implementer`
subagent implements (Sonnet 5.5, medium effort). Your job is to resolve the target, dispatch the
subagents with enough context, handle git/`gh`, and drive the loop.

Always talk to the user in English. Keep the running commentary short — one line per phase.

---

## Phase 0 — Resolve the target

The fault to fix is `$ARGUMENTS`.

If `$ARGUMENTS` is empty, take the target from the **immediately preceding message in this
conversation** — typically the user's last message, or the last report you produced (very often a
`pr-auditor` report or a failing test run).

- Restate in one line what you understood the fault to be, then continue.
- Only stop and ask if the previous message contains no identifiable fault, or contains several
  unrelated ones and it is not obvious which is meant.

Build a **fault dossier** now — you will paste it verbatim into every subagent prompt, because
subagents start with a fresh context and see none of this conversation:

- Description of the fault in the user's own words
- Error messages, stack traces, failing test names, reproduction steps — verbatim, not summarised
- File paths, symbols or endpoints already mentioned
- Anything already ruled out or already tried

## Phase 1 — Pre-flight

Verify and **stop** if something is wrong:

- You are inside a git repository with a configured remote (`git remote -v`).
- The current branch is not `main`, `master` or `develop`. If it is, propose a branch named
  `fix/<slug-kebab-case>` derived from the fault and ask before creating it.
- Uncommitted changes unrelated to this fix: list them and ask whether to continue, stash, or stop.
- `gh auth status`. If `gh` is missing or unauthenticated, warn and continue in degraded mode:
  commit and push still run, PR creation is skipped, and the audit falls back to
  `git diff <base>...HEAD` instead of a PR number.

Create a TodoWrite list for the loop so progress stays visible.

## Phase 2 — Plan (`fix-planner`, Opus 5.5, medium)

Delegate to the `fix-planner` subagent. Its prompt must contain:

- The fault dossier verbatim
- On iteration 2 and later: the **new** `pr-auditor` findings verbatim, plus a short summary of
  what was already changed in previous iterations and why, so it does not re-tread ground
- The repository's stack, if you already know it, to save the planner discovery time

Do not modify any file during this phase. When the plan comes back, print it to the user.

If the planner returns **Blocked** instead of a plan, stop the loop: show the user what it looked
at and what information it needs, and wait. Do not dispatch the implementer.

## Phase 3 — Implement (`fix-implementer`, Sonnet 5.5, medium)

Delegate to the `fix-implementer` subagent. Its prompt must contain the plan **verbatim** plus the
fault dossier. Make explicit that it must implement the plan and nothing else.

When it returns:

- If it reports build or test failures it could not resolve, treat those failures as new findings
  and go back to Phase 2. This counts as an iteration.
- If it reports that the plan was wrong or unimplementable, go back to Phase 2 with its reasoning
  attached. This also counts as an iteration.

## Phase 4 — Commit and push

- Stage only the files the implementer touched.
- One commit per iteration, conventional format: `fix(<scope>): <what was fixed>`, imperative
  mood, en-GB. If the repo has its own commit convention or an issue key in the branch name,
  follow that instead.
- `git push -u origin HEAD`. Never force-push. Never push to `main`, `master` or `develop`.

## Phase 5 — Pull request

- `gh pr view --json number,state,isDraft,url` on the current branch.
- If a PR already exists: reuse it, note its number, do not change its title or state.
- If there is none: `gh pr create --draft` with a title matching the commit subject and a body
  containing the fault description, the plan summary and a "pending audit" note. Draft on purpose
  — the PR should not reach a reviewer before the audit is clean.

```bash
gh pr create --draft --title '<title>' --body-file - <<'EOF'
<body>
EOF
```

- The body goes through a quoted heredoc, so backticks and `$` are passed literally. Never put the body inside double quotes.
- The title is single-quoted. Write any `'` in it as `'\''`.
- The `EOF` terminator must not appear on a line of its own in the body.

## Phase 6 — Audit (`pr-auditor`)

Invoke the `pr-auditor` subagent against the PR number, or against `git diff <base>...HEAD` when
there is no PR. Then classify what it returns:

| Finding | Action |
| :--- | :--- |
| 🔴 Critical | New iteration — back to Phase 2 |
| 🟡 Warning | New iteration — back to Phase 2 |
| 🔵 Suggestion | Reported to the user, **not** fixed automatically |
| 📦 Dependencies | Reported to the user, not fixed automatically |
| No 🔴 and no 🟡 (🔵/📦 may remain) | Loop ends — go to Phase 7 |

The loop ends on the critical and warning counts from the auditor's `Result:` line alone; suggestions never trigger another iteration, so `CLEAN` is not required.

Each new iteration carries **only the new findings** into the planner, never the whole report
again.

### Loop guards

- **Maximum 3 iterations.** On reaching the limit, stop, keep the branch and PR as they are, and
  report what is still open.
- **Repeated finding**: if the auditor raises a finding that is substantially the same as one from
  a previous iteration, stop the loop. It is either a false positive or a genuine disagreement, and
  another automatic round will not resolve it — hand it to the user with both sides described.
- **Growing diff**: if an iteration touches files well outside the original fault, stop and ask.

## Phase 7 — Final report

In English, briefly:

- What was fixed and the root cause, in one or two lines
- How many iterations it took and what each one resolved
- Files touched and the PR link
- Anything left open: suggestions not applied, dependency notes, findings that hit a loop guard
- A reminder that the PR is still a draft

---

## Hard rules

- Never weaken, skip or delete a test to make the suite green. If a test is genuinely wrong, say so
  and stop.
- Never expand the fix beyond the fault: no opportunistic refactors, no unrelated formatting,
  no dependency bumps that the plan does not require.
- Never force-push, never rewrite published history, never touch protected branches.
- Never mark the PR ready for review — that decision is the user's.
- If any subagent is missing (`fix-planner`, `fix-implementer`, `pr-auditor`), stop and say which
  one, rather than doing its job in the main context.