---
name: openspec-git-commit
description: "MANDATORY skill that activates whenever the OpenSpec apply phase begins. Triggers: /opsx:apply runs, openspec-apply-change is referenced or active, `openspec instructions apply` is invoked, or the user asks to implement/apply/execute/build out an OpenSpec change. Wraps the vanilla apply step 6 implementation loop: commits each completed task and verifies commits when all tasks are done."
version: 1.0.0
priority: high
disable-user-invocation: true
allowed-tools: Bash(git:*), Bash(openspec:*)
---

# OpenSpec Git Commit

## Mission

Wrap the vanilla `openspec-apply-change` implementation loop with a one-commit-per-task discipline and an end-of-apply verification. The vanilla skill (verified source: `OpenSpec/src/core/templates/workflows/apply-change.ts`) owns:

- Change selection.
- `openspec status --change <name> --json` and `openspec instructions apply --change <name> --json`.
- Reading context files (`proposal.md`, `spec.md`/`specs/**`, `design.md`, `tasks.md`).
- Progress display.

This skill adds, around vanilla step 6 (the implementation loop):

- A single "planning artifacts" commit for any uncommitted `openspec/` files present at apply start.
- One commit per task, immediately after that task's checkbox is flipped to `[x]`.
- An end-of-apply verification: every checked task has a matching commit, tree is clean, before archive is suggested.

---

## Behavioral contract

### 1. Branch pre-flight (before implementing anything)

Run `git branch --show-current`.

- If the current branch matches `^(feat|bugfix|refactor|docs|chore)/.+`:
  - Extract `<prefix>` = text before the first `/`.
  - Extract `<change-name>` = text after the first `/`.
  - These two drive every commit message on this branch.
- If the current branch is the main branch (`main` or `master`):
  - **Stop.** Report that apply is starting on the main branch with no feature branch. Ask the user:
    - **Run the `openspec-git-branch` flow now** — retroactive branch creation. Run that skill's steps 2–7 for the current change's name. Working-tree changes (proposal/spec/design/tasks files already written) are preserved across `git checkout -b` by standard git behavior — they carry onto the new branch. After the branch is created, continue at step 2 below.
    - **Proceed without per-task commits** — the user explicitly opts out; this skill does nothing further. (Not recommended.)
- If the current branch is anything else (a non-main, non-feature branch):
  - **Warn** and ask the user whether this is the intended change branch. Only proceed on explicit confirmation. If confirmed, extract `<prefix>`/`<change-name>` from the branch name as in the first bullet (best-effort; the prefix may not match the mapping — use whatever precedes the first `/`).

### 2. Planning-artifacts commit

Before the first task is implemented, check for uncommitted OpenSpec planning files:

```
git status --porcelain -- openspec/
```

- If **non-empty** (there are uncommitted proposal/specs/design/tasks files for this change): stage and commit them as a single planning commit.
  - `git add openspec/`
  - Commit message: `<type>(<change-name>): add planning artifacts` where `<type>` is the branch prefix (the same `<type>` is used for every commit on this branch).
- If **empty**: skip silently — the tree was already clean (e.g. the `openspec-git-branch` skill or the vanilla workflow already committed them, or the user committed manually).

### 3. Wrap the vanilla apply loop

The vanilla `openspec-apply-change` workflow runs its implementation loop (step 6) exactly as it would without this skill — this skill does not reimplement change selection, instruction loading, or progress display. It adds the following hook **after each task is marked `- [x]` in the tasks file, before the next task is started**:

1. `git add -A` — the tree was clean at apply start (after step 2), so this sweeps only the current task's edits plus the checkbox flip in `tasks.md`.
2. Commit with subject:
   - `<type>(<change-name>): task X.Y <task text>` when the checkbox has a numeric id.
   - `<type>(<change-name>): <task text>` when the checkbox has no numeric id.
   - `X.Y` is parsed from the checkbox line: `- [x] 1.2 Add toggle` → `task 1.2 Add toggle`.
   - `<task text>` is the task's first line, truncated so the full subject stays ≤ 72 characters (truncate at the last whitespace boundary at or before character 72; if the first word itself is longer, hard-cut at 72).
3. If the commit fails with "nothing to commit" (the task produced no file changes beyond an already-committed state — e.g. it was only a documentation/verification step whose outputs were already committed):
   - Skip the commit.
   - **Note it** for the final report (step 4) under a "no-op tasks" list.

Do not batch multiple tasks into a single commit. One task, one commit.

### 4. End-of-apply verification (when every task is `[x]`, before suggesting archive)

Run this verification once the vanilla workflow reports all tasks complete.

1. Detect the main branch the same way `openspec-git-branch` step 3 does: `main`, else `master`, else ask the user.
2. `git log <main-branch>..HEAD --oneline` — list every commit on the feature branch since it diverged from main.
3. Parse the tasks file for every checked task id. For each checked task with a numeric id `X.Y`, assert that a commit exists whose subject contains the literal `task X.Y`.
4. `git status --porcelain` must be empty.

Report:

```
<N> tasks, <M> task commits, tree clean ✓
```

plus the full commit list from `git log <main-branch>..HEAD --oneline`.

If any checked task with a numeric id lacks a matching commit, **or** the tree is dirty:

- List the gaps explicitly (missing commit for `task X.Y`, or uncommitted files from `git status --short`).
- **Stop.** Ask the user how to reconcile:
  - **Commit leftovers now** — `git add -A` and commit `<type>(<change-name>): finish incomplete tasks` (or similar). Re-run the verification.
  - **Amend** the relevant task commit. (Allowed here only at the user's explicit request — see Guardrails.)
  - **Investigate** — do nothing further; let the user debug.

Do **not** silently proceed to archive suggestions when the verification fails.

### 5. Guardrails

- **Never commit directly on the main branch.** If apply starts on main, step 1 stops and offers the `openspec-git-branch` flow first.
- **Never `git push`.** Local-only.
- **Never auto-trigger `/opsx:archive` or `openspec archive`.** After step 4 passes, you may *suggest* archiving; the user runs it.
- **Never amend existing commits** — except when the end-of-apply verification fails and the user explicitly chooses "amend" in step 4. Unprompted amends are forbidden.
- **One commit per task, no batching.** Planning artifacts are the only non-task commit.
