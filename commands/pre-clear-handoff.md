---
allowed-tools: Read, Edit, Write, Glob, Grep, Bash(git status:*), Bash(git log:*), Bash(git diff:*), Bash(git show:*), Bash(git rev-parse:*)
description: Pre-/clear handoff — record everything valuable from current context into the given md file so a fresh session can pick up cleanly
argument-hint: <path to handoff/plan md file>
---

# Pre-Clear Handoff

Target file: `$ARGUMENTS`

If empty, ask user for path. If file doesn't exist, ask whether to create it or pick a different path. Do not silently invent a path.

## Goal

User is about to run `/clear`. After clear, the new session has zero memory of this conversation. Everything that matters has to live in durable artifacts — primarily the target md file, secondarily the memory system and the codebase itself. Your job: **extract every load-bearing piece of context from this session and write it where a fresh session can find it.**

If you do this right, the next session reads the target file once and is fully caught up.

## Workflow

### 1. Survey what was done in this session

Without modifying anything, look at:

- The full conversation history available to you
- `git status` — uncommitted work, staged but uncommitted, untracked files
- `git log --oneline -20` — what landed
- `git diff` of working tree vs HEAD, and HEAD vs `origin/<branch>` if branch tracks a remote
- Open todos / tasks (TaskList) if any
- Any plan / handoff file already at `$ARGUMENTS` — read it to know what's already recorded

Build a mental list of:
- What changed in the repo
- What was decided in conversation
- What was tried and rejected (negative knowledge — easy to lose)
- What is in progress / half-done
- What the user said about preferences, constraints, deadlines, who owns what
- What was learned about the codebase that's not obvious from reading the code

### 2. Classify each item

For each piece of context, decide where it belongs:

- **Target md file (`$ARGUMENTS`)** — feature/project-specific state: progress, decisions, in-progress work, session log entries, next-session focus, blocking unknowns resolved this session, plan divergences taken.
- **Memory system** (`memory/`) — durable cross-session knowledge: user preferences confirmed/corrected, project context that outlives this feature, reference pointers to external systems, codebase invariants surfaced by user feedback. Use the four memory types (user / feedback / project / reference) per the auto-memory rules.
- **Code or its surrounding files** — anything that should live as a comment near the code, a CLAUDE.md addition, or a test. (Note as a recommendation; do not auto-edit code in this command.)
- **Discard** — ephemeral chatter, repeated info, anything already obvious from `git log` / source.

### 3. Update the target md file

Open `$ARGUMENTS`. Find the right sections to update:

- **Status / current state** — overwrite with the latest snapshot. Include branch, latest commit SHA, MR / PR link if any, test status.
- **Per-ticket / per-task progress** — tick off what shipped, mark in-progress.
- **Session log** — append a dated entry summarizing this session: what landed, what was caught in review, force-pushes, decisions made, anything surprising. Use absolute dates (resolve "today" / "yesterday" against the current date in context).
- **Blocking unknowns** — strike resolved ones (with the resolution noted), add new ones surfaced this session.
- **Next session focus** — concrete, actionable. What file to open first, what ticket to start on, what command to run.
- **Open questions / risks** — refresh.

Rules for the target file:
- Edit in place. Do not blow away existing content.
- Keep the file authoritative — if conversation contradicts the file, the latest decision wins; note the override in the session log.
- Stay terse. Future sessions read this every time.
- Resolve relative dates ("Thursday", "tomorrow") to absolute (YYYY-MM-DD) using the current date in context.

### 4. Update memory if anything qualifies

If anything in step 2 belongs in `memory/`, write it now per the auto-memory rules. Don't write project-specific implementation details there — that's what the target file is for. Memory is for durable, cross-session value.

Specifically capture:
- **feedback** — user corrections or confirmations of approach surfaced this session (include the **Why** and **How to apply**)
- **user** — newly learned facts about the user's role / preferences / responsibilities
- **project** — durable project context (ownership, deadlines, motivations) that survives this feature
- **reference** — external resources / dashboards / tickets the user pointed to

### 5. Surface anything that should land in code

Things like "this invariant should be a comment near `foo()`" or "this convention belongs in `<package>/CLAUDE.md`" — do **not** apply them in this command. List them in the final report so the user can decide whether to do them before /clear.

### 6. Verify the handoff is sufficient

Read the updated `$ARGUMENTS` end to end as if you were a fresh session. Ask:

- Can I tell what branch I'm on?
- Can I tell what shipped vs what's in flight?
- Can I tell what to do next, concretely?
- Are there decisions I'd have to re-litigate because they're not recorded?
- Are there gotchas the user mentioned in chat that aren't in the file?

If any answer is "no", go back and fix the file.

## Output

Single response after the writes are done, ≤ 400 words. Structured:

```
## Handoff target
<path>

## Updates written to target file
- bulleted summary of what changed in $ARGUMENTS

## Memory files written / updated
- list with one-line description, or "none"

## Recommended code-side edits (NOT applied)
- list with file path + what to add, or "none"

## Verification — fresh-session readiness
- branch / commit / MR state recorded? ✅/❌
- next-session focus concrete? ✅/❌
- decisions captured? ✅/❌
- gotchas captured? ✅/❌

## Safe to /clear?
yes / no, with one-line reason
```

## Rules

- Edit `$ARGUMENTS` and memory files only. Do **not** modify code, do **not** commit, do **not** push.
- Do not invent new state. Only record what actually happened.
- If you find yourself recording something that's already obvious from `git log` or source, skip it.
- If the user is mid-decision (e.g. about to choose between two approaches), record both options and the trade-off — do not unilaterally pick.
- If something genuinely should not be cleared yet (e.g. uncommitted critical work), say "Safe to /clear: no" and explain.
