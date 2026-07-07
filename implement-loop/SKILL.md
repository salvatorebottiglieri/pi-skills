---
name: implement-loop
description: |
  Process a batch of work items: implement each via a TDD subagent, review
  via a reviewer subagent, fix until clean, align docs, then create a PR.

  Use when the user says "implement these issues", "process this queue",
  "agentic loop", or passes a list of items to implement. User-invoked only.
disable-model-invocation: true
---

# Implementation Loop

A disciplined agentic loop for implementing work items one at a time:
**implement → review → fix → align docs → PR**. Every change goes through a
subagent; the main agent never edits code during the loop.

## 0. Setup

- Read the items from wherever the user's issue tracker lives (the user
  tells you where — GitHub Issues, GitLab, Linear, a local file, etc.).
- Identify the spec source per item (the item body IS the spec).
- Check for dependencies between items — order them so no item depends on
  unmerged work. Resolve dependencies sequentially; run independent items in
  parallel with `worktree: true`.

## 1. Implement — delegate to a subagent

Spawn a `worker` subagent with `context: "fresh"` (`async: true`) with:

- The full item body (title + description) as the task.
- `Use /tdd (test-driven development).` at the top of the task.
- The relevant file paths and code context the subagent needs (current
  function signatures, API, existing test patterns).
- A clear list of files the subagent **may** modify vs **must not** touch.
- The acceptance criteria from the item.

Do not implement yourself. Do not edit files. The subagent is the sole writer.

**Always pass `context: "fresh"`** so the worker starts with a clean context
window and receives only the information you explicitly include in the task
string. Inheriting the parent session's context (the default for `fork` agents)
causes the worker to see unrelated messages, files, and decisions that can
drift its output — or worse, skip making edits because it hallucinates the
changes are already applied.

## 2. Review — delegate to a reviewer

When the implement subagent completes, spawn two `reviewer` subagents with
`context: "fresh"` in sequence (or parallel if the tool allows):

- **Standards review**: full diff against HEAD + code smell baseline (Fowler,
  Refactoring ch.3). Report per-file findings.
- **Spec review**: compare the diff against the item description verbatim.
  Report gaps, scope creep, and wrong implementations.

Both reviewers are read-only. They do not edit code.

**Always pass `context: "fresh"`** so each reviewer sees only the diff and
the review criteria you provide, not the entire chat history. A reviewer that
inherits the parent session may confuse its mandate with the implementer's or
the loop manager's.

## 3. Fix — delegate, never fix directly

If either review finds actionable issues:

1. **Spawn a fresh `worker` subagent** with `context: "fresh"` (`async: true`)
   to fix them. Pass the review findings **verbatim** as task context.
2. In the task, tell the subagent *which files* to change and *what
   specifically* to fix (quote the findings). Include the current diff for
   context.
3. **Be explicit in the fix task**: tell the subagent to edit files and run
   tests. A fix subagent that inherits context may plan without acting —
   `context: "fresh"` prevents this by forcing you to put every instruction
   in the task string.
4. When the fix subagent completes, go back to **step 2** (re-review).
4. Exit the loop only when both reviews return **zero actionable findings**.

**Hard rule**: the main agent never edits code during the fix loop. Not
  formatting, not comments, not one-line renames. Every change goes through a
  subagent. The only exception: findings classified as purely mechanical
  (whitespace, typos, comments). Behavioural, structural, or logic changes
  always delegate.

## 4. Align docs

When all items pass review and before creating the PR, check whether the
code changes need documentation updates:

1. **Identify affected docs**: grep the diff for changes that touch public
   APIs, CLI flags, config formats, environment variables, data schemas,
   architecture decisions (ADRs), or anything documented externally.
2. **Check project conventions**: look for `CONTEXT.md`, `README.md`,
   `docs/ARCHITECTURE.md`, `docs/adr/`, or any `*_docs/` directory that maps
   to the changed code.
3. **Delegate doc updates**: if the diff changes something documented, spawn
   a `worker` subagent with `context: "fresh"` to:
   - Read the relevant docs and the code diff.
   - Update docs to match the new behaviour.
   - Skip docs that are still accurate — no speculative rewrites.
4. **No news is good news**: if nothing documented changed, skip this phase
   entirely. The PR step proceeds directly.

## 5. PR

When all items pass review and docs are aligned:

1. `git checkout -b <branch-name>` (descriptive, e.g. `feat-<issue-number>`)
2. `git add` the changed files (only what belongs to the task)
3. `git commit -m "..."` with a conventional commit message referencing the items
4. `git push origin <branch-name>`
5. Create a PR using the project's standard tooling (ask the user how).
6. Close/resolve each completed item in the issue tracker.

## Dependency resolution

- **Sequential**: if item B depends on item A's code, implement A → review A
  → fix A → align docs A → merge A (or at least commit A on a shared base)
  → then implement B. No worktree needed for sequential.
- **Parallel**: if items are independent, run each through the full
  implement/review/fix/align-docs loop in its own git worktree (`worktree: true`
  on the subagent call). Merge them in dependency order.

## Completion criterion per item

The loop is done with an item when:
- All acceptance criteria from the item description are met
- All tests pass (existing + new)
- A reviewer (Standards + Spec) reports zero actionable findings
- Docs are aligned with the change (or confirmed unnecessary)
- The item is referenced in a PR or commit
