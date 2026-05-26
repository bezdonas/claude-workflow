---
description: Create a fully-developed plan file in CLAUDE_PLANS/ from a Linear issue or text description. Stops before any code or git side effects.
argument-hint: Linear ID/URL or task description, optionally + topic name
---

# Plan Creator

Target plans directory: `/Users/ramilaliev/devprojects/tutteo/CLAUDE_PLANS/`

Input: `$ARGUMENTS`

## Goal

Produce a single plan markdown file at `<plans dir>/<TOPIC>.md` that is a fully-developed engineering plan: phases, files to touch, acceptance criteria, branch/MR layout, cross-cutting invariants. **Every uncertain decision is explicitly flagged with `❔` for `/grill-me` to drill into.** The file is the long-lived source of truth across all subsequent sessions on this work.

This command produces **no git side effects** (no branch, no commit) and **writes no code**. It only writes the plan file.

## Workflow

### 1. Parse arguments + detect mode

Inspect `$ARGUMENTS`:

- **Linear mode** — argument contains a Linear issue identifier (regex `[A-Z]+-\d+`) or a `linear.app/...` URL.
- **Standalone mode** — no Linear reference; `$ARGUMENTS` is a plain task description.
- **Topic override** — if user supplied an explicit `ALL_CAPS_UNDERSCORE` token at the end, use it as the filename stem.

If `$ARGUMENTS` is empty, ask user (single AskUserQuestion) what to plan and stop until answered.

### 2. Fetch Linear context (Linear mode only)

Use the Linear MCP to fetch:

- Issue title, description, current status, assignee, team
- Comments
- If the issue belongs to a project, also fetch the project + sibling sub-issues so the plan can lay out the full ticket map

For standalone mode, skip this step — `$ARGUMENTS` is the source.

### 3. Derive topic name

If user supplied `TOPIC`, use it. Otherwise generate a **concise** `ALL_CAPS_UNDERSCORE` name that captures the essence of the work (e.g. `A_TEMPO`, `EXPORT_TO_MIDI`). Prefer short — 1-3 words.

Check if `<plans dir>/<TOPIC>.md` already exists:

- If exists → ask user (AskUserQuestion): overwrite, append numeric suffix (e.g. `A_TEMPO_2`), or abort.
- If not → proceed.

### 4. Codebase exploration

Read enough of the codebase to write a real plan, not a guess:

- Identify which packages / repos the work touches
- Read each touched package's `CLAUDE.md` (if present)
- Find similar existing patterns (`Grep` for analogs)
- Read the shape of key files the plan will integrate with
- Note conventions: commit prefix, branch naming, test format

Keep exploration scoped — depth of one similar feature is plenty. `/grill-me` and `/pre-investigate-plan` will go deeper.

### 5. Identify uncertainties

As you draft the plan, every time you make a decision you are not certain about, **stop and flag it** rather than committing to it in the plan. Examples of uncertainty:

- Schema field name / shape
- Whether a particular approach matches package conventions
- Which of two layers should own a responsibility
- Whether a behavior is intentional or incidental in existing code
- Forward-compatibility implications
- Boundary cases (empty / overlap / collision / undo)

Each becomes a numbered entry under **Blocking unknowns** with: the question, options considered, your recommendation, and what evidence would decide it.

### 6. Write the plan

Write to `<plans dir>/<TOPIC>.md` using the template below. Fill the sections substantively — this is not a skeleton. Leave `❔` only on actual uncertainties.

#### Template

```markdown
# <Topic title-case> — Implementation Plan

## Plan-as-progress-tracker (MANDATORY)

**This file is the single source of truth across Claude sessions.** Every session reads it on entry; every session updates it before exit. A cleared session must be able to reconstruct current state by reading this file alone.

Discipline:

- Update the **Status** section below at the start and end of every session.
- Tick checkboxes in **Cross-cutting checklist** as items complete.
- Mark commits in **MR / branch plan** sections as `(done)` once committed (and `(pushed)` once pushed).
- When a decision in the plan turns out wrong during implementation, update the relevant section in place — do not leave stale guidance.
- Append session-end notes to **Status > Session log** with a short summary of what happened that session.

Failure to keep this file current = next session works on stale assumptions.

## Push policy

NEVER push without explicit user "push" confirmation. Force-push only with `--force-with-lease`.

## Fixup policy

Review fixes and bug fixes land as `git commit --fixup=<sha>` against the original feature commit, then autosquash + force-push-with-lease before push. NEVER push standalone `fix:` commits onto a still-in-review feature branch.

For amend-fixups that need a new subject or body, use the `amend!` first-line trick to keep autosquash matching:

```
write to /tmp/msg.txt:
  amend! <original subject>

  <new subject>

  <new body>

GIT_EDITOR="cp /tmp/msg.txt" git commit --fixup=amend:<sha>
```

Without the `amend!` first line, autosquash treats it as a regular commit.

## Status

**Project state:** <one-line current state — initially: planning, no code yet>

**Current session focus:** planning

**Next session focus:** Run `/grill-me <path>` then `/pre-investigate-plan <path>` to resolve Blocking unknowns, then start work via `/work-plan <path>`.

**Local-only working state (NEVER commit):** none yet

**Quickstart for next session:**
1. Read this file top-to-bottom.
2. <repo-specific git status / branch confirm commands — fill from MR / branch plan section>
3. <build prerequisites, if any>

**Gotchas:** <empty initially; fill as discovered>

**Blocking unknowns:**

<Numbered list. Each entry:>
<1. ❔ <question>>
<   - Options: <A / B / C>>
<   - Recommendation: <X, because ...>>
<   - Evidence needed: <code path / user decision / external doc>>

### Session log

- <YYYY-MM-DD> (planning) — Plan authored from <Linear ID | text description>. <N> blocking unknowns surfaced.

### Per-ticket progress

<If Linear project with sub-issues, table:>

| Ticket | Status | Final SHAs | Pushed |
|--------|--------|------------|--------|
| <ID>   | pending | —         | no     |
| ...    | pending | —         | no     |

<If single-issue or standalone, omit table; track via Cross-cutting checklist instead.>

<If Linear:>
Linear project: <URL if applicable>
Assignee / Lead / Team: <from Linear>
Project window: <dates from Linear>

## Feature summary

<2-4 paragraphs. What the feature is, why it exists, the cross-cutting semantic rule (if any), high-level shape. For Linear mode, distilled from issue description + comments. For standalone, distilled from $ARGUMENTS.>

## Repos / packages

<List of touched repos + packages. Note any cross-repo dependencies.>

## Ticket map + dependency graph

<If multi-ticket, a code block diagram showing parent → child relationships. If single-ticket, omit.>

## Sequencing

### Phase 1 — <Phase name> (blocking | parallelizable)

**<TICKET-ID> (N)** — `<repo>/<package>` — <one-line summary>

- <Bullet of concrete change: new file path, new exported symbol, change to existing file>
- <... fill in detail for each file>
- **Tests** (`<test path>`):
  - <test case 1>
  - <test case 2>
- **AC:**
  - <Acceptance criterion 1>
  - <Acceptance criterion 2>
- **Changeset:** <minor | patch | none> (`<pkg>`)

**<TICKET-ID> (N+1)** — ...

### Phase 2 — ...

<Mark every uncertain detail inline with ❔ + reference to Blocking unknowns entry number.>

## Cross-cutting checklist

Items every phase must respect. Tick as completed.

- [ ] <Invariant 1 — e.g. "every new direction-type wired into ClearMeasureColumn cleanup">
- [ ] <Invariant 2 — e.g. "snapshot regeneration via `pnpm update-tests --grep ...`">
- [ ] <Invariant 3 — e.g. "package CLAUDE.md additions if new convention introduced">
- [ ] Tests + lint + tsc clean on every commit
- [ ] No push without user "push" confirmation
- [ ] All review fixes land as fixups (not standalone fix: commits)

## MR / branch plan

### Repo: `<repo>`

- **Branch:** `<feat|fix|chore>/<id-list>/<short-desc>` (e.g. `feat/ed-816-817-a-tempo`)
- **Base:** `<develop | other base ref + sha if pinned>`
- **Commits planned:**
  1. `<type>(<pkg>): <subject> [<TICKET-ID>]` — <one-line scope>
  2. `<type>(<pkg>): <subject> [<TICKET-ID>]` — ...
- **MR:** opened as draft via `glab mr create -d ...` after first push. Title: `<type>: <description> [<TICKET-IDs>]`.

### Repo: `<repo-2>` (if multi-repo)

...

## Open questions (non-blocking, for follow-up)

<Questions the human may want to revisit but that don't block code work. E.g. "do we also want to add MIDI export here, or separate ticket?">
```

### 7. Print summary + next steps

After write:

- Print the plan path
- Print the count of `❔` Blocking unknowns
- Suggest the next commands in user's workflow:
  - `/grill-me <path>` — interview to resolve open questions
  - `/pre-investigate-plan <path>` — code-level verification
  - `/criticize-plan <path>` — adversarial review
  - `/pre-clear-handoff <path>` — before clearing context
  - `/work-plan <path>` — once plan is locked

## Rules

- **No git operations.** No branches, no commits, no pushes.
- **No code edits.** Only the plan file is written.
- **No empty sections.** If a section doesn't apply (e.g. no Linear project), omit it instead of leaving a placeholder header.
- **No fake confidence.** Every uncertain decision becomes a `❔` Blocking unknown, not a stated decision.
- **Respect plan path.** Hardcoded `/Users/ramilaliev/devprojects/tutteo/CLAUDE_PLANS/`. Do not write elsewhere.
- **Stay terse.** Future sessions read this every time.
