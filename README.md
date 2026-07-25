# openspec-git-flow

A standalone bundle that extends [OpenSpec](https://github.com/Fission-AI/OpenSpec) with a git branch / commit / merge workflow. It ships as parallel auto-activating `SKILL.md` files dropped into a coding agent's skills directory, plus three `context:` lines merged into the project's `openspec/config.yaml`. No OpenSpec core changes; state flows between skills through git itself (the current branch name carries `<prefix>/<change-name>`).

## What it enforces

On every OpenSpec change, the bundle enforces:

1. **Change start** — verify the repo is on the main branch; if not → stop and ask.
2. **Clean check** — verify main is clean; if dirty → stop and ask what to do.
3. **Branch creation** — if clean → create branch `<prefix>/<change-name>`, where `<prefix>` is derived from the change name.
4. **Per-task commits** — during apply, one commit per completed task, message `<type>(<change-name>): task X.Y <task text>`.
5. **End-of-apply verification** — verify every checked task has a matching commit and the tree is clean before archive is suggested.
6. **Archive merge** — on `/opsx:archive` (after the vanilla archive workflow finishes): commit the archive move, `git checkout main`, `git merge --no-ff`, delete the feature branch. **Local only — never push.**

## Mechanism

Three auto-activating skills plus a config-merge. State flows via the branch name.

| Skill | Activates on | Owns |
|---|---|---|
| [`openspec-git-branch`](skills/openspec-git-branch/SKILL.md) | new change start (`/opsx:new`, `/opsx:propose`, `/opsx:ff`, `openspec new change`) | main-branch + clean checks, prefix derivation, feature-branch creation |
| [`openspec-git-commit`](skills/openspec-git-commit/SKILL.md) | apply phase (`/opsx:apply`, `openspec instructions apply`) | planning-artifacts commit, one commit per task, end-of-apply verification |
| [`openspec-git-merge`](skills/openspec-git-merge/SKILL.md) | archive (`/opsx:archive`, `openspec archive`) | archive commit, `--no-ff` merge to main, feature-branch deletion |

Each skill's `description` frontmatter is trigger-laden so a skill loader that matches on description activates it at the right phase. The vanilla OpenSpec workflows still own proposal/spec/design/tasks authoring, change selection, instruction loading, and the archive move — this bundle only inserts the git gate before change start, the commit hook around the apply loop, and the merge after archive.

**Compatibility note:** if a future OpenSpec release renames the vanilla skills or commands (`openspec-new-change`, `openspec-propose`, `openspec-ff-change`, `openspec-apply-change`, `openspec-archive-change`, `openspec-bulk-archive-change`), the `description` trigger lists in each `SKILL.md` must be updated to match. This coupling is by design — the bundle wraps vanilla workflows rather than replacing them.

---

## ⚡ Install / Update

> **Prerequisites:** [OpenSpec](https://github.com/Fission-AI/OpenSpec) initialized in your project (`openspec init`) and a supported AI coding agent.

Copy the prompt below and paste it directly into your AI coding agent. It will download, install, update, and configure everything automatically.

> **This same prompt works for both first-time installation and updates.** Run it anytime to get the latest version. If you already have the skills installed, it overwrites them in place.

```text
Install/Update OpenSpec Git-Flow Skills

Your task is to install or update the OpenSpec git-flow skills and configuration
from the GitHub repository https://github.com/sh1ny/openspec-git-flow into the
current project.

Follow these steps in order. Ask the user when indicated — do not assume answers.

Step 1 — Verify OpenSpec is set up

Check if an `openspec/` directory exists in the current working directory.
- If it does NOT exist: inform the user that OpenSpec is not set up in this project.
  Suggest they run `openspec init` first. STOP here — do not continue.
- If it exists: proceed to Step 2.

Step 2 — Determine skills directory

Detect the coding agent environment by checking which of these directories exist
in the current working directory:
- `.claude/skills/` (Claude Code)
- `.opencode/skills/` (OpenCode)
- `.agents/skills/`
- `.cursor/skills/` (Cursor)
- `.antigravity/skills/` (Antigravity)

If exactly one exists, use it as the default. If multiple exist, list them and ask
the user which to use. If none exist, ask the user for a path (suggest creating
`.claude/skills/` or the one matching their agent).

Ask the user: "Where should I install the skills?" and present the detected
default (if any). Let them confirm or provide a custom path.

Step 3 — Download from GitHub

Download the repository https://github.com/sh1ny/openspec-git-flow to a temporary
location. Use whatever method is available to you (`git clone --depth 1`,
`gh repo clone`, `curl`, direct file download, etc.). You only need the `skills/`
directory, `VERSION` file, and `openspec/config.yaml` from the repository.

Step 4 — Install skills and VERSION file

From the downloaded repository:
1. Copy all three `openspec-git-*` directories inside `skills/` to the skills
   directory confirmed in Step 2:
   - skills/openspec-git-branch/
   - skills/openspec-git-commit/
   - skills/openspec-git-merge/
   Each is a directory containing a SKILL.md — copy the entire directory recursively.
2. Copy the `VERSION` file to `openspec/.git-flow/VERSION` in the current project.
   Create the `openspec/.git-flow/` directory if it does not exist. (This lets
   later installs report what version was copied; the bundle does no auto-update
   check.)

- If the target skills directory does not exist, create it.
- If a skill directory already exists at the target, overwrite its contents
  (this is an update).
- Track which skills were newly installed vs updated (the directory existed before).

Step 5 — Merge config.yaml

Check if `openspec/config.yaml` exists in the current project:

- If it does NOT exist: copy the downloaded `openspec/config.yaml` from the
  repository as-is to `openspec/config.yaml` in the current project.
- If it DOES exist: merge the bundle's three `context:` lines into the existing
  config:
  - For the `context` field: append any of the three MANDATORY lines from the
    bundle that are not already present in the existing context. The three
    lines are:
      MANDATORY: Invoke the 'openspec-git-branch' skill immediately when a new OpenSpec change is started (/opsx:new, /opsx:propose, /opsx:ff, openspec-new-change, openspec-propose, openspec-ff-change) — BEFORE 'openspec new change' runs or any change file is created.
      MANDATORY: Invoke the 'openspec-git-commit' skill immediately when the OpenSpec apply phase begins (/opsx:apply, openspec-apply-change, 'openspec instructions apply').
      MANDATORY: Invoke the 'openspec-git-merge' skill immediately after the OpenSpec archive workflow completes (/opsx:archive, openspec-archive-change, 'openspec archive').
  - Do NOT remove or modify any existing content in the user's config.yaml.
    Only add what is missing.

Step 6 — Clean up

Remove the temporary download directory.

Step 7 — Summary report

Provide a brief summary with:
- The skills directory used
- How many skills were installed (noting how many were new vs updated)
- Whether config.yaml was created, merged, or left unchanged
- The VERSION that was copied

Keep the summary concise — no line-by-line breakdown.

Recommend the user to restart their code editor or coding agent for the skills to take effect.
```

---

## ⚡ First Steps After Install

Once installed and your agent restarted, try it on a real change:

1. Start a new change with `/opsx:propose` or `/opsx:new` — the `openspec-git-branch` skill gates first: you must be on `main` (or `master`) with a clean tree, then it creates the feature branch.
2. Run apply (`/opsx:apply`) — each task you check off `[x]` becomes its own commit on the feature branch.
3. Archive (`/opsx:archive`) — after the vanilla archive move, the feature branch is merged into `main` with `--no-ff` and deleted.

The fastest way to feel the value is to run it on something you're actually building.

---

## 🔧 Manual Installation

If you prefer not to use the AI prompt:

1. Clone the repository:

   ```bash
   git clone --depth 1 https://github.com/sh1ny/openspec-git-flow.git /tmp/openspec-git-flow
   ```

2. Copy the skills to your agent's directory and the VERSION file to `openspec/.git-flow/` (example for Claude Code):

   ```bash
   cp -r /tmp/openspec-git-flow/skills/openspec-git-branch .claude/skills/
   cp -r /tmp/openspec-git-flow/skills/openspec-git-commit .claude/skills/
   cp -r /tmp/openspec-git-flow/skills/openspec-git-merge .claude/skills/
   mkdir -p openspec/.git-flow && cp /tmp/openspec-git-flow/VERSION openspec/.git-flow/VERSION
   ```

3. Merge the three `context:` lines into your `openspec/config.yaml`:

   ```bash
   cat /tmp/openspec-git-flow/openspec/config.yaml
   # Manually append the three MANDATORY lines under `context: |` to your existing config.yaml
   # (do not remove or modify any existing content)
   ```

4. Clean up:

   ```bash
   rm -rf /tmp/openspec-git-flow
   ```

5. Restart your coding agent so the skills load.

---

## 📖 Behavior reference

### Branch prefix mapping

The prefix is derived from the first `-`-separated token of the change name (lowercased):

| Leading token | Prefix |
|---|---|
| `add`, `feat`, `feature`, `implement`, `introduce` | `feat/` |
| `fix`, `bugfix`, `hotfix`, `patch`, `resolve` | `bugfix/` |
| `refactor`, `restructure`, `cleanup` | `refactor/` |
| `docs`, `doc`, `document` | `docs/` |
| `chore`, `bump`, `deps`, `ci`, `build` | `chore/` |
| anything else | `feat/` (the skill tells the user which prefix was assumed) |

### Commit message formats

| When | Format |
|---|---|
| Apply start, uncommitted `openspec/` files present | `<type>(<change-name>): add planning artifacts` |
| Each task checked `[x]` | `<type>(<change-name>): task X.Y <task text>` |
| Task with no numeric id checked `[x]` | `<type>(<change-name>): <task text>` |
| After archive move | `<type>(<change-name>): archive change` |
| Merge to main | `merge: <change-name>` |

`<type>` is the branch prefix (`feat`, `bugfix`, `refactor`, `docs`, `chore`). One commit type per branch — the planning-artifacts and archive commits reuse the branch's prefix so the log stays uniform and the end-of-apply verification is trivially countable. `<task text>` is the task's first line, truncated to keep the full subject ≤ 72 characters.

### Merge flow

After the vanilla archive workflow completes:

1. Commit the archive move (and any synced spec edits) on the feature branch: `<type>(<change-name>): archive change`.
2. `git checkout <main-branch>` (main, else master, else ask).
3. `git merge --no-ff <prefix>/<change-name> -m "merge: <change-name>"`.
4. On conflict: stop, list conflicted files, ask the user to resolve manually or `git merge --abort`. Never auto-resolve.
5. `git branch -d <prefix>/<change-name>` (lowercase `-d`; never `-D`).
6. Report the merge commit and: `Local only — push <main-branch> when ready.`

### Main-branch detection

`main`, else `master`, else ask the user. No remote-HEAD probing — the workflow is local-only.

---

## 🗑️ Uninstall

To remove the bundle:

1. Delete the three skill directories from your agent skills dir:
   ```bash
   rm -rf .claude/skills/openspec-git-branch .claude/skills/openspec-git-commit .claude/skills/openspec-git-merge
   ```
2. Remove the three MANDATORY `context:` lines (the ones naming `openspec-git-branch`, `openspec-git-commit`, `openspec-git-merge`) from `openspec/config.yaml`. Leave all other content intact.
3. Optionally remove `openspec/.git-flow/VERSION`.
4. Restart your coding agent.

---

## 💡 Inspiration

The build and install model — parallel auto-activating `SKILL.md` files plus a `context:` merge into `openspec/config.yaml` — is inspired by [`openspec-plus`](https://github.com/sudokar/openspec-plus), which applies the same pattern to enforce disciplined proposal/spec/design/tasks authoring. This bundle applies the pattern to a different concern (git workflow) and is independent of it; the two can be installed side by side.
