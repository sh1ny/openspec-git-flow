---
name: openspec-git-branch
description: "MANDATORY skill that activates whenever a new OpenSpec change is started. Triggers: /opsx:new, /opsx:propose, or /opsx:ff runs; openspec-new-change, openspec-propose, or openspec-ff-change is referenced or active; `openspec new change` is about to be invoked; or the user asks to start, create, scaffold, or propose a new OpenSpec change. Runs BEFORE any change files are created: verifies the repo is on the main branch and clean, then creates the feature branch <prefix>/<change-name>."
version: 1.0.0
priority: high
disable-user-invocation: true
allowed-tools: Bash(git:*), Bash(openspec:*)
---

# OpenSpec Git Branch

## Mission

Run as a gate before any new OpenSpec change is scaffolded. Before `openspec new change` (or the equivalent vanilla `openspec-new-change` / `openspec-propose` / `openspec-ff-change` workflow) creates a single file, this skill:

1. Confirms the working tree is a git repo and is on the main branch.
2. Confirms the main branch is clean.
3. Derives `<prefix>/<change-name>` from the change name.
4. Creates and checks out the feature branch.

The vanilla change-creation workflows still own proposal/spec/design/tasks authoring and the `openspec` CLI calls — this skill only inserts the git-branch gate in front of them. All change artifacts are then written on the feature branch.

---

## Behavioral contract

Execute these steps in order. Stop and ask the user at every indicated point — never proceed silently on a failed check.

### 1. Pre-flight git check

Run `git rev-parse --is-inside-work-tree`.

- If it **fails**: this is not a git repo. **Stop.** Report the error and ask whether to proceed without git branching. The recommended option is **abort** (the rest of the git-flow bundle depends on a repo). Do not run any `openspec` command before this check passes.
- If it **succeeds**: continue.

### 2. Resolve the change name

- If the user already gave a kebab-case name (matches `^[a-z0-9]+(-[a-z0-9]+)*$`), use it verbatim.
- If they gave only a description, derive a kebab-case name the same way the vanilla `openspec-new-change` skill does (verified source: `OpenSpec/src/core/templates/workflows/new-change.ts`) — lowercase, spaces to hyphens, drop non-alphanumerics, collapse repeats. Example: "add user authentication" → `add-user-auth`.

**Do not create anything yet.** This step only produces the `<change-name>` string used by steps 5–7.

### 3. Branch check

Run `git branch --show-current` to get the current branch. Determine the main branch in this order:

- `git rev-parse --verify main` succeeds → main branch is `main`.
- else `git rev-parse --verify master` succeeds → main branch is `master`.
- else **stop** and ask the user which branch is the main branch.

If the current branch is **not** the main branch: **stop.** Report the current branch name and ask the user how to proceed. Offer:

- **Switch to main** — only offer this option when `git status --porcelain` is empty (a dirty tree would carry across the switch and corrupt the branch). When chosen: `git checkout <main-branch>` then continue at step 4.
- **Abort** — do nothing.

Never proceed on a non-main branch silently.

### 4. Clean check

Run `git status --porcelain`.

- If output is **non-empty**: **stop.** Show the full `git status --short` output and ask the user how to proceed. Offer:
  - **Commit the changes first** — let the user/user workflow commit on main, then re-run this skill from step 4.
  - **Stash them** — `git stash push -u -m "openspec-git-branch: pre-<change-name>"`, then continue. Tell the user the stash ref so it can be restored later.
  - **Abort** — do nothing.
- If output is **empty**: continue.

Never create the feature branch while main is dirty unless the user explicitly chose to proceed after seeing the status.

### 5. Derive prefix

Take the first `-`-separated token of `<change-name>` (lowercased) and map it:

| Leading token | Prefix |
|---|---|
| `add`, `feat`, `feature`, `implement`, `introduce` | `feat/` |
| `fix`, `bugfix`, `hotfix`, `patch`, `resolve` | `bugfix/` |
| `refactor`, `restructure`, `cleanup` | `refactor/` |
| `docs`, `doc`, `document` | `docs/` |
| `chore`, `bump`, `deps`, `ci`, `build` | `chore/` |
| anything else | `feat/` |

When the leading token matches none of the rows and the `feat/` fallback is used, tell the user in one line which prefix was assumed (e.g. `Assuming feat/ (no matching token for "sync").`).

### 6. Collision check

Run `git rev-parse --verify <prefix>/<change-name>`.

- If it **succeeds** (the branch already exists): **stop.** Report that the branch already exists and ask the user:
  - **Switch to it to resume the change** — `git checkout <prefix>/<change-name>`, then skip to step 8 (the change is already in progress; the feature branch exists).
  - **Pick a different change name** — return to step 2 to resolve a new name.
  - **Abort** — do nothing.
- If it **fails** (branch does not exist): continue.

### 7. Create branch

Run `git checkout -b <prefix>/<change-name>`. Announce:

```
On branch <prefix>/<change-name> (from <main-branch>).
```

### 8. Hand off

Proceed with the workflow that triggered this skill (`/opsx:propose`, `/opsx:new`, or `/opsx:ff`) using the resolved `<change-name>`. The scaffold and all artifacts created by the vanilla workflow are now written on the feature branch.

If step 6 handed off to an existing branch, resume the vanilla change workflow in place — do not re-scaffold.

---

## Guardrails

- **Never `git push`.** This bundle is local-only.
- **Never commit at this stage.** Commits belong to the apply phase (`openspec-git-commit`) and the archive phase (`openspec-git-merge`).
- **Never create the branch when any pre-flight, branch, or clean check fails** without an explicit user choice.
- **Always run this skill before any `openspec new change` invocation**, regardless of the entry point (`/opsx:new`, `/opsx:propose`, `/opsx:ff`, or a direct CLI call).
- **Never assume the main branch.** Detect it (step 3) or ask.
