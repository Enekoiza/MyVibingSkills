---
name: planning
description: Generates a detailed action plan for the requested change, without implementing anything until the user explicitly confirms.
argument-hint: [change to plan]
model: claude-opus-5-5
effort: medium
disable-model-invocation: true
---

# Planning Mode

The user is requesting the following change: $ARGUMENTS

If "$ARGUMENTS" is empty or contains only whitespace, ask what change they want to plan and stop there.

Always talk to the user in English, including the confirmation question and the plan headings.

## Phase 1 — Research (read-only)

- Explore the relevant code: project structure, affected files,
  dependencies, existing tests, and repo conventions (CLAUDE.md, linters,
  existing patterns).
- For broad searches across many files, delegate to the `Explore` subagent
  and keep only its conclusions, so the main context stays small.
- Check whether the folder is a git repository and, if so, the current branch
  (read-only, e.g. `git branch --show-current`).
- FORBIDDEN in this phase: creating, editing, or deleting files, running
  commands with side effects (installing packages, migrations, git commit,
  etc.). Read and search only.

## Phase 1.5 — Clarification gate

Before writing any plan, explicitly decide whether you must ask the user
something. This check is mandatory, not optional.

Ask when any of these holds after your research:
- The scope is unclear or too vague to plan (e.g. "improve the app").
- There are several reasonable approaches, libraries, or patterns with
  different trade-offs.
- The change touches data, schemas, or public APIs, or could break existing
  behavior.
- Acceptance criteria or expected behavior are missing.

How to ask:
- Ask only what the code and repo could not answer and what would change the
  plan. Decide everything else with sensible defaults.
- Send one numbered batch of questions. For each, offer the options and mark
  a recommended one.
- End the turn with the questions only — no plan and no confirmation request.

After the user answers, run this gate again. If new ambiguities appeared, ask
again; otherwise continue to Phase 2.

If nothing blocks you, go straight to Phase 2 and list the defaults you chose
under **Assumptions**.

## Phase 2 — Plan

Scale the plan to the change: a one-file tweak gets a few lines, a
cross-cutting change gets every section. Omit sections that would be empty.

**Goal**: one sentence summarizing what will be achieved.

**Context found**: 2-4 lines on the relevant current state
(what exists today, what constrains the approach).

**Assumptions**: defaults you chose without asking the user, one line each.

**Chosen approach**: which strategy you're following and why. If you
discarded a reasonable alternative, mention it in one line.

**Steps**: numbered list where each step indicates:
- Affected file(s) and whether it's created/modified/deleted
- What changes in it (description, NOT the full code)
- Commands to run, if any (installs, migrations, scripts)

**Impact and risks**:
- Possible side effects (breaking changes, data, performance)
- What tests will be added or updated
- Points of uncertainty, if any

**Out of scope**: what will NOT be touched, to bound expectations.

**Branch**:
- Not a git repository: say so; no branch will be created.
- Current branch is `main`, `master` or `develop`: give the name of the new
  branch that will be created.
- Otherwise: work continues on the current branch (`<current-branch>`).

Keep the plan scannable. No diffs or full files in this phase.

## Phase 3 — Confirmation

Whenever you present a plan (initial or revised), end with this question:

> Do you confirm I should implement this plan as is? Reply "go ahead" to
> implement, or tell me what you'd like to adjust.

Strict rules:
- Do NOT implement anything in the same turn as the plan, even if the change
  is trivial or the user sounded urgent.
- Only proceed on explicit, unambiguous approval ("go ahead", "yes,
  implement", "confirmed", or equivalents). For ambiguous responses ("ok I see",
  "interesting", a question), clarify or answer, and ask for confirmation again.
- If the user asks for changes, present the full REVISED plan, starting with
  a short "Changes from the previous version" list, and ask for confirmation
  again. Never implement a revision without re-confirming.

## Phase 4 — Implementation (only after confirmation)

- If the folder is a git repository:
  - Before touching any file, check the working tree. If there are any
    uncommitted changes, STOP and ask how to proceed.
  - Then check the current branch:
    - If it is `main`, `master` or `develop`, create and switch to a new branch:
      `git checkout -b <type>/<short-kebab-case-slug>`, where `<type>` is
      `feature`, `fix`, or `chore` depending on the change, and the slug
      summarizes the plan's goal (e.g. `feature/users-endpoint-pagination`).
    - If it is already another branch (for example, one created by a previous
      planning session), continue working on that branch. Do NOT create a new
      branch and do NOT ask about it.
- If it is not a git repository, skip the git steps and implement directly.
- Implement exactly what was approved, step by step.
- If during implementation you discover something that forces a deviation
  from the plan (an unexpected file, a test that reveals a design problem), STOP,
  explain the deviation, and ask for confirmation on the adjustment before continuing.
- When implementation is finished, if the project has a test suite, use the
  `test-runner` subagent to verify it still passes before considering the task
  closed. If it reports failures or changes that require your decision, present
  them and wait before closing the task. If the `test-runner` subagent is not
  available, say so instead of running the suite yourself in the main context.
  If the project has no tests, say so.
- Summarize what was done, mapping it against the plan's steps, including the
  test-runner result and the branch used (if any), and indicate how
  to verify it manually if applicable.
