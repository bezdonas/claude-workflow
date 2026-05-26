---
description: Execute a plan file's Next-session-focus. Resume-style — one focused chunk per invocation. Updates the plan in place. Never pushes without explicit user confirmation.
argument-hint: Path to plan md file (defaults to most-recently-modified in CLAUDE_PLANS/)
---

# Plan Executor

Plans directory: `/Users/ramilaliev/devprojects/tutteo/CLAUDE_PLANS/`

Target plan: `$ARGUMENTS`

## Goal

Read the plan, locate the **Status > Next session focus** line, do that work, run tests, commit (as new commit OR fixup), update the plan file (Status + Session log + Per-ticket progress), then **stop and wait for explicit user "push"** before any push.

The plan file is the source of truth. Every session reads it on entry and updates it before exit.

## Inviolable rules

1. **Never push without explicit user "push" confirmation.** Force-push only with `--force-with-lease`. This is not negotiable.
2. **Review/bug fixes land as fixups**, not standalone `fix:` commits. Use `git commit --fixup=<sha>` against the original feature commit. Autosquash before push. For commits needing new subject/body, use `amend!` first-line trick.
3. **Plan file always updated before exit.** Status, Session log, Per-ticket progress, MR/branch plan SHAs — all kept current. Stale plan = next session works on stale assumptions.
4. **No schema changes (DB/API) without pre-approval in plan.** If the work needs an unapproved schema change, stop and ask user.
5. **No scope expansion.** Do what Next-session-focus says. Defer everything else to a new plan entry or new ticket.

## Workflow

### 0. Resolve plan path

- If `$ARGUMENTS` empty → list `.md` files in plans dir sorted by mtime, propose the newest, confirm with user (AskUserQuestion) before continuing.
- If `$ARGUMENTS` does not exist → stop, ask user.

### 1. Read plan top-to-bottom

Read the entire file. Extract:

- Status block — current focus, next focus, local-only working state, quickstart
- Push policy + fixup policy
- Blocking unknowns — note any still-open (`❔`) ones; if Next-session-focus is gated on one, refuse to proceed and tell user to `/grill-me` first
- Per-ticket progress table — pin the SHAs you may need for fixup targets
- MR / branch plan — pin the branch name(s) per repo

Build a TodoWrite list of concrete sub-steps based on Next-session-focus.

### 2. Quickstart preamble

Run the plan's **Status > Quickstart for next session** steps. Typical:

- `cd <repo>` for each touched repo
- `git status` — confirm clean / expected dirty state
- If Local-only working state mentions stash, `git stash list` → `git stash pop` if expected entries present
- `git log --oneline -<N>` per repo to confirm branch tip matches plan's recorded SHAs (if tip diverges from plan, stop and reconcile before doing any work)
- Build prerequisites if noted (`pnpm --filter ... build`)

### 3. Branch handling

For each repo touched by Next-session-focus:

- Check current branch: `git branch --show-current`
- If matches plan's MR/branch plan branch → continue
- If branch doesn't exist locally → `git checkout -b <name>` per plan (off the documented base)
- If on wrong branch → stop, ask user; do not switch silently

### 4. Confirm focus

Print to user: "Next session focus per plan: <quoted>. Proceeding."

Wait for one explicit confirmation if the focus involves cross-package or destructive changes. For routine continuation, proceed.

### 5. Implement

Do the focused chunk:

- Read every file the plan references before editing
- Edit / write code per plan's Sequencing section
- Follow package CLAUDE.md conventions
- **Schema-change gate** — if the implementation reveals an unapproved DB/API schema change, **stop**. Surface to user. Add to Blocking unknowns. Do not proceed.
- Write tests alongside implementation
- Run tests incrementally (per-package): `pnpm --filter <pkg> test`
- Run lint / tsc per-package as you go
- All tests must pass before commit

### 6. Commit

Determine commit shape from plan + diff:

**New ticket / new functionality** → new commit:
```
<type>(<pkg>): <subject> [<TICKET-ID>]

<body — what + why if non-obvious>
```

`<type>` ∈ `feat | fix | chore | docs | test | refactor` per conventional commits. `<TICKET-ID>` matches Linear ID from plan.

**Fix to existing ticket** (review feedback, bug found during impl, missing wiring):
```
git commit --fixup=<original-SHA from Per-ticket progress>
```

**Amend-fixup needing new subject/body** (e.g. behavior change that warrants rewriting the commit message of the original):
```bash
# Write to /tmp/msg.txt:
#   amend! <original subject>
#
#   <new subject>
#
#   <new body>

GIT_EDITOR="cp /tmp/msg.txt" git commit --fixup=amend:<original-SHA>
```

The `amend!` first line is REQUIRED for autosquash to match. Without it, autosquash creates a regular commit.

Stage with `git add <specific files>`. Avoid `git add -A` / `git add .` to keep local-only files (e.g. pnpm-workspace.yaml overrides, dev config) out.

### 7. Update plan file in place

Open the plan. Edit:

- **Status > Current session focus** → completed work this session (or blank if rolling forward)
- **Status > Next session focus** → next concrete chunk (read user's mind from plan + commit history if obvious; else "user to decide")
- **Status > Session log** → append a new entry:
  ```
  - <YYYY-MM-DD> (<short subject>) — <2-5 sentences: what landed, key decisions, SHAs, tests passing, what's queued next>
  ```
  Use absolute dates (resolve "today" against context current date).
- **Per-ticket progress table** → tick what landed this session, update SHAs if a rebase shifted them
- **Cross-cutting checklist** → tick boxes as items complete
- **Blocking unknowns** → strike through resolved ones (note resolution + date), add new ones surfaced
- **Local-only working state** → update if any local-only files added/removed

Save. Do not blow away existing content.

### 8. Print summary + suggest next

After plan update, print to user:

```
## Summary

<What was done this session in 3-5 bullets>

## Plan updates

<What was written to <plan path>>

## Next steps (user-driven)

- `/local-review` — code review the new commit(s) before push
- `git rebase -i --autosquash <base>` — collapse fixups before push (only if fixups were created this session)
- "push" — say push to authorize `git push --force-with-lease`
- `/pre-clear-handoff <plan>` — if context filling, run before /clear
```

### 9. Push gate (user-initiated only)

Do NOT push unless the user explicitly says "push" / "yes push" / "go push" / similar clear authorization. When they do:

- For each repo modified:
  - `git rebase -i --autosquash <base>` first if fixups exist (use `GIT_SEQUENCE_EDITOR=:` to skip editor)
  - Stash any local-only files first (per plan's Local-only working state list)
  - `git push --force-with-lease` (NEVER plain `--force`)
  - `git stash pop` to restore local-only files
- After push:
  - Update plan: mark commits as `(pushed)`, update SHAs that shifted from rebase, update MR / branch plan section with new tips
  - Update **Per-ticket progress** Pushed column to `yes`
  - Append a session log entry noting the push

If user does not say push, the session ends with commits local. Plan reflects this state.

## Gotchas to encode

- Stash workflow during rebase: `git stash push -m "..." -- <specific files>` (preserve specifics), rebase, force-push, `git stash pop`.
- Biome reformats JSON / generated files on pre-commit. Let it normalize; don't fight.
- `pnpm-workspace.yaml` overrides + per-package `link:` deps in `package.json` are local-only and must NEVER be committed. If unsure, `git diff --staged` before commit.
- Some files mix committed + uncommitted state (e.g. dep bump + link override on same package.json). Use `git add -p` to stage hunks selectively, or `git stash` the uncommitted portion before commit.
- Use `glab` (not `gh`) for all GitLab operations.
- MR creation happens via `glab mr create -d` after the first push. The MR title should be `<type>: <description> [<TICKET-IDs>]`. This is user-driven; don't auto-create the MR.

## Memory invariants to respect

These live in `memory/` and apply automatically; surface relevant ones in user output when triggered:

- `feedback_amend_fixup_workflow.md` — preserve `amend! <subject>` first line for autosquash matching
- `feedback_no_conversation_labels_in_code.md` — never let grill-session labels (Q1/D3/M1/T1) leak into code comments, spec titles, or commit bodies
- Any package-specific feedback memories (e.g. `feedback_adagio_cleanup_invariant.md`, `feedback_toolset_editable_set_gate.md`) for the packages you're touching

## Rules

- Plan is the contract. Don't extend scope beyond Next-session-focus.
- Never push without explicit user authorization.
- Always update the plan file before exit.
- Use fixups for review/bug fixes, not standalone fix commits.
- Use `glab` for GitLab ops.
- Conventional commits + `[TICKET-ID]` suffix.
- Force-push only with `--force-with-lease`.
