---
name: openspec-git-merge
description: "MANDATORY skill that activates whenever the OpenSpec archive phase runs. Triggers: /opsx:archive or /opsx:bulk-archive runs, openspec-archive-change or openspec-bulk-archive-change is referenced or active, `openspec archive` is invoked, or the user asks to archive/finalize/complete an OpenSpec change. Runs AFTER the vanilla archive workflow completes: commits the archive move, merges the feature branch into the main branch with --no-ff, and deletes the feature branch."
version: 1.0.0
priority: high
disable-user-invocation: true
allowed-tools: Bash(git:*), Bash(openspec:*)
---

# OpenSpec Git Merge

## Mission

Run only **after** the vanilla `openspec-archive-change` workflow (verified source: `OpenSpec/src/core/templates/workflows/archive-change.ts`) has fully completed — including its completion checks, spec sync, the `changeRoot` → `archive/` move, and its own summary. This skill:

1. Commits the archive move (plus any synced spec edits) on the feature branch.
2. Merges the feature branch into the main branch with `--no-ff`.
3. Deletes the feature branch.

The vanilla archive workflow owns everything about archiving (selection, completion checks, spec sync, directory move, summary). This skill owns only the git operations that follow.

---

## Behavioral contract

### 1. Wait for vanilla archive

Do not begin until the vanilla `openspec-archive-change` workflow reports success.

For **bulk archive** (`openspec-bulk-archive-change`, verified in `OpenSpec/src/core/templates/workflows/bulk-archive-change.ts`): let the vanilla workflow complete all of its individual archives first, then run steps 2–6 **once** (one commit, one merge, one branch deletion per archived change — see step 3 for how multiple changes are handled).

If the vanilla archive workflow failed or was aborted, **stop** — do not run any git operations.

### 2. Branch pre-flight

Run `git branch --show-current`.

- If on the main branch (`main`/`master`): there is no feature branch in use for this change. **Skip all git steps** and report: `No branch to merge — archive completed on <main-branch>.`
- Otherwise: record `<prefix>/<change-name>` from the current branch name (text before/after the first `/`). If the branch name does not match the archived change's name, **warn** and ask the user before continuing (the branch may belong to a different change).

### 3. Archive commit

After the vanilla move, the working tree has uncommitted changes: the change directory was moved into `archive/`, and any synced spec edits are pending.

For a single archive:

- If `git status --porcelain` is **non-empty**:
  - `git add -A`
  - Commit: `<type>(<change-name>): archive change` where `<type>` is the branch prefix.
- Then assert `git status --porcelain` is empty. If it is not, **stop** and report the leftover files.

For **bulk archive** with multiple changes on the same feature branch (rare but possible — bulk-archive usually operates per-change): if multiple changes were archived in one vanilla run and they share a branch, fold them into a single archive commit: `<type>(<change-name>): archive change` (use the primary change name; list the others in the commit body if helpful). If they are on separate branches, run steps 3–6 once per branch.

### 4. Merge

Detect the main branch the same way `openspec-git-branch` step 3 does: `main`, else `master`, else ask the user.

Run:

```
git checkout <main-branch>
git merge --no-ff <prefix>/<change-name> -m "merge: <change-name>"
```

- If the merge completes cleanly: continue to step 5.
- If the merge **stops on conflicts**: **stop.** List the conflicted files from `git status`. Ask the user:
  - **Resolve manually then `git commit`** to finish the merge.
  - **`git merge --abort`** to return to a clean main and abandon the merge.
  - Do **not** auto-resolve. Do not pick a side.

### 5. Delete branch

After a successful merge:

```
git branch -d <prefix>/<change-name>
```

- Use `-d` (lowercase), **never `-D`**. `-d` refuses to delete an unmerged branch; it succeeds here precisely because the branch is now merged.
- If `-d` refuses (should not happen after a clean `--no-ff` merge): **stop** and report — do not fall back to `-D`.

### 6. Summary

Show:

- The merge commit subject + short hash from `git log -1 --oneline`.
- That the feature branch `<prefix>/<change-name>` was deleted.
- The line: `Local only — push <main-branch> when ready.`

---

## Guardrails

- **Never push.** Local-only.
- **Never force-delete** (`git branch -D` is forbidden).
- **Never merge before the vanilla archive workflow has fully completed** — including its spec-sync verification. The archive move and spec edits must be committed (step 3) before the merge.
- **Never auto-resolve merge conflicts.** Stop and ask.
