---
name: audit
description: Deep read-only code audit for bugs, performance problems, security issues and improvements. Reports findings in the conversation, colour-coded by severity. Never writes files or modifies code. Invoke manually with /audit.
argument-hint: "[path | 'diff' | 'staged' | 'since <ref>' | empty = whole project]"
model: claude-opus-5-5
effort: high
disable-model-invocation: true
allowed-tools: Read, Grep, Glob, Bash(git diff *), Bash(git log *), Bash(git status *), Bash(git ls-files *), Bash(grep *), Bash(sort *), Bash(uniq *), Bash(head *), Bash(true)
---

# /audit — deep code audit

## Repository context

- Branch and dirty state: !`git status --short --branch || true`
- Recent commits: !`git log --oneline -15 || true`
- Files changed in the last 20 commits: !`git log --name-only --pretty=format: -20 | grep -v '^$' | sort | uniq -c | sort -rn | head -30 || true`

If these lines show a git error (not a git repository, or no commits yet), there is no git context: audit a path or the whole project by reading the files directly, and for `diff`, `staged` or `since <ref>` say that git is required and stop.

## Scope

`$ARGUMENTS` selects what to audit:

| Argument | Scope |
| --- | --- |
| empty | The whole project. Start with the files listed above as most-changed, then the entry points, then the rest. |
| a path | That file or directory, recursively. |
| `diff` | Uncommitted changes only (`git diff`), plus enough surrounding code to judge them. |
| `staged` | Staged changes only (`git diff --cached`), plus surrounding context. |
| `since <ref>` | Everything changed since `<ref>` (`git diff <ref>...HEAD`). |

If the scope is larger than you can read carefully, say so up front, audit the
highest-risk subset first (entry points, auth, data access, concurrency,
anything touched recently), and list at the end what you did not reach.

Write the report in English.

## Method

Work in this order. Do not skip step 1 — a finding you cannot trace to real
code is a false positive, and false positives are worse than misses here.

1. **Map before judging.** Identify the language, framework, layering, and the
   project's own conventions. A pattern that is idiomatic for this codebase is
   not a finding just because you would write it differently.
2. **Read the code paths, not just the files.** Follow a request or job from
   entry point to persistence at least once before reporting on it.
3. **Verify each candidate finding.** Re-read the exact lines. Check whether a
   guard, attribute, middleware, or caller already handles the case. Drop it if
   it is already handled.
4. **Rank by blast radius**, not by how easy it is to describe.

## What to look for

### Correctness and bugs
- Null / undefined / default-value dereferences, and nullable returns treated as non-null.
- Off-by-one errors, wrong boundary conditions, inverted comparisons.
- Unhandled error paths: swallowed exceptions, empty catch blocks, ignored
  return codes, `catch` that logs and continues into an invalid state.
- Async problems: unawaited tasks, fire-and-forget without error handling,
  `async void`, sync-over-async deadlock patterns, missing cancellation tokens.
- Concurrency: shared mutable state without synchronisation, non-atomic
  check-then-act, race conditions, deadlock-prone lock ordering.
- Resource leaks: undisposed connections, streams, timers, subscriptions,
  event handlers, file handles.
- State and lifetime mistakes: captured stale closures, singletons holding
  request-scoped state, mutable static state.
- Logic that contradicts the surrounding code's stated intent (comments, names,
  tests) — flag the mismatch, and say which side you believe is wrong.

### Performance
- N+1 queries and per-item round trips inside loops.
- Queries without an index-friendly predicate, `SELECT *` on wide tables,
  missing pagination, sorting or filtering in memory that belongs in the query.
- Repeated work that should be hoisted, memoised, or cached; recomputation
  inside hot loops.
- Unnecessary allocations, large object copies, string concatenation in loops,
  boxing in tight paths.
- Blocking I/O on request or UI threads; missing streaming for large payloads.
- Front-end specifics: avoidable re-renders, missing memoisation on expensive
  children, unkeyed lists, unbounded state in context, effects that fire every
  render, oversized bundles from a single heavy import.
- Missing timeouts, retries without backoff, or retries that amplify load.

### Security
- Injection risks: string-built SQL, unparameterised queries, shell or path
  construction from user input, unsafe deserialisation.
- Missing or wrong authorisation checks, especially on endpoints that sit
  beside protected ones.
- Hardcoded secrets, connection strings, tokens, keys.
- Missing input validation at trust boundaries; over-posting / mass assignment.
- Output encoding gaps that allow XSS; unsafe HTML rendering.
- Sensitive data in logs, error responses, or exception messages.

### Maintainability and design
- Duplicated logic that has already drifted between copies.
- Functions or classes doing several unrelated jobs; leaked layer boundaries.
- Misleading names, dead code, unreachable branches.
- Missing or weak tests around the risky code you just flagged.
- Configuration or magic values that should be named or externalised.

## Severity rules

Assign exactly one severity per finding, using these definitions:

- **CRITICAL** — causes incorrect behaviour, data loss or corruption, a crash
  in a reachable path, or exposes a security hole. Someone should fix it before
  the next release.
- **WARNING** — a real defect or a meaningful performance cost, but bounded:
  it degrades rather than breaks, or the bad path is unlikely but reachable.
  It should be fixed, but it does not block.
- **SUGGESTION** — no defect. Design, readability, or a minor optimisation that
  a reasonable reviewer could decline.

If you hesitate between two levels, choose the lower one and say why in the
finding. Do not inflate severity to make the report look thorough.

## Output rules

- **Never create, write, or edit any file.** No report file, no patch, no
  scratch notes. The entire output goes in the conversation.
- **Never modify code.** Show the fix; do not apply it.
- Number findings sequentially across the whole report (1, 2, 3…) so the user
  can refer to them.
- Cite every finding as `path/to/file.ext:line` (or `:startLine-endLine`).
- Quote the smallest fragment that makes the problem visible — a few lines, not
  the whole function.
- Give the fix as concrete code, not as advice ("consider using…" is not a fix).
- One finding per issue. If the same bug appears in five places, report it once
  and list the other locations underneath it.
- If a category has nothing, write `No findings.` under it and move on.
- Do not pad. A short report with five real findings beats thirty weak ones.

## Report format

Open with two lines: what you audited, and how much of it you actually read.

Then the findings, grouped by severity in this order, using these exact markers:

### 🔴 CRITICAL

````
🔴 1. `src/Api/UserController.cs:42` — Short title
   What happens, and why it matters. 2–3 lines. Name the path that reaches it.

   ```csharp
   // the offending lines
   ```

   **Fix:**
   ```csharp
   // the corrected lines
   ```
   Also at: `src/Api/OrderController.cs:88`, `src/Api/CartController.cs:19`
````

### 🟡 WARNING

Same structure.

### 🔵 SUGGESTION

Same structure, but the fix may be a one-line description instead of code when
the change is trivial.

### Summary

End with exactly two lines:

```
🔴 N · 🟡 N · 🔵 N
Not covered: <areas you did not reach, or "none">
```

Then offer, in one sentence, to fix a specific finding by number if the user
wants — but do not start fixing anything unless they ask.
