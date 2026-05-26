---
allowed-tools: Bash(glab mr view:*), Bash(glab mr list:*), Bash(glab mr diff:*), Bash(git log:*), Bash(git show:*), Bash(git rev-parse:*), Bash(rg:*), Bash(fd:*), Read, Edit, Write, Glob, Grep, TodoWrite
description: Mine a finished Linear project's MRs for reusable patterns and write them into relevant CLAUDE.md files
argument-hint: Linear project URL or slug, optional --scope=<heading>, --repos=<list>, --dry-run
---

# Project Pattern Miner

Goal: given finished Linear project, investigate every related MR across all touched repos, distill **reusable patterns** for next similar work, write findings into relevant CLAUDE.md files.

Output is documentation, not code review. Mine past work for repeatable shape — files always touched, recurring steps, dual-system bridges, cross-repo sequencing, gotchas surfaced in review.

## Arguments

Parse `$ARGUMENTS`:
- Linear project URL or slug (required)
- `--scope=<heading>`: section heading in CLAUDE.md. Default: derive from project name (e.g. "Adding a New Notation", "Migrating X to Y"). Ask user if ambiguous.
- `--repos=<csv>`: restrict scan to listed repos under workspace root
- `--dry-run`: print proposed edits, skip writes

If no project arg: ask user.

---

## Workflow

### Step 0: Todo list

TodoWrite. Track all steps. Mark done as you go.

---

### Step 1: Pull Linear project + issues

Use Linear MCP:
- `mcp__claude_ai_Linear__get_project` — description, goals, scope notes
- `mcp__claude_ai_Linear__list_issues` filtered by project — every issue with title, description, status, attachments, comments

Confirm project finished (most issues Done/Released). If not finished, warn user, ask to continue.

Build list: `(issue, branch_name, mr_links[])`. Branch pattern often `feature/<ticket>/...`.

Derive default `--scope` heading from project name + description. State derived heading to user before proceeding (one line).

---

### Step 2: Discover MRs across repos

For each issue, collect MR links from:
- Linear `attachments` / external links
- Fallback: `glab mr list --search "<ticket-id>"` per repo under workspace root. Workspace root = parent of cwd or git toplevel's parent. Detect repos = direct subdirs containing `.git`.

Group MRs by repo. Record: MR ID, repo, title, target branch, merge commit SHA, files changed list.

Skip closed-without-merge MRs unless their discussion holds useful lesson.

---

### Step 3: Parallel MR analysis

Launch **one Sonnet subagent per repo** (single message, multiple Agent calls). Each agent gets:
- Repo absolute path
- MR list for that repo
- Project description from Linear
- Derived scope heading
- Instruction: "Building knowledge base for repeating this kind of work. Extract reusable patterns, not project-specific names/values."

Tools per agent: `glab mr view <ID>`, `glab mr view <ID> --comments`, `glab mr diff <ID>`, `git log --follow`, `git show <sha>`. Read MR description AND discussion comments — gotchas live in review threads.

Each agent returns structured report:

```
## Repo: <name>

### Files / directories always touched
- path — why

### Recurring code patterns
- pattern — file(s) — short snippet if shape non-obvious

### Dual-system / migration bridges (if applicable)
- where split lives
- how feature registers in each system
- removal plan signals

### Cross-repo coupling
- exported symbols/types consumed elsewhere
- changeset / version bump conventions observed

### Sequencing / commit structure observed
- ordered MR titles, one-line each
- which repo merges first; what publishes to internal registry; what consumes

### Gotchas / lessons
- things needing multiple commits or force-pushes
- review-comment-driven changes
- anti-patterns rejected during review
```

---

### Step 4: Synthesize cross-repo flow

Combine repo reports:
- End-to-end build sequence (repo order, publish→consume edges)
- Canonical checklist for repeating this work
- Files-to-touch keyed by repo
- Cross-check workspace `MEMORY.md` and root `CLAUDE.md` — don't duplicate existing content

---

### Step 5: Locate target CLAUDE.md files

For each repo with findings, pick most specific existing CLAUDE.md (Glob `**/CLAUDE.md` per repo, pick deepest dir matching changed files). Prefer specific over root.

Workspace root CLAUDE.md gets cross-repo flow + sequencing.

If no good existing fit in a repo, append new section to that repo's root CLAUDE.md.

---

### Step 6: Compose edits

For each target CLAUDE.md, prepare additive section under derived `--scope` heading:

```markdown
## <scope heading>

### Files to touch
- `path/to/file.ts` — what to change

### Steps
1. ...

### Dual-system / bridges
- ...

### Gotchas
- ...
```

Rules:
- Patterns only — strip project-specific identifiers, feature names, values
- No code dumps; short snippets only when shape non-obvious
- Terse — CLAUDE.md read every turn
- **No MR/commit/SHA references.** They rot (squash, force-push, repo migration), cost tokens every turn, and encourage copy-paste over understanding. If an example helps, point at a `grep`-able file/function name instead — survives moves, points at code reader can read now.
- No "Pattern derived from <project>" preamble — patterns stand on their own merit, not on provenance

If section heading already exists in target CLAUDE.md: merge intelligently — augment lists, don't duplicate bullets, preserve existing wording when equivalent.

---

### Step 7: Write edits

Default behavior: **write edits** with Edit tool, one CLAUDE.md at a time. No user confirmation between files.

If `--dry-run`: skip writes, print diffs instead.

After writes: re-read each modified file, sanity check formatting (no broken markdown, no orphan headings).

---

### Step 8: Output summary

```
Linear project: <name>
Scope heading: <heading>
Issues analyzed: N
MRs analyzed: M across <repos>
CLAUDE.md files updated: <list with paths>

Key patterns extracted:
- ...

Open questions worth confirming:
- ...
```

Ambiguity (e.g. "is this bridge code about to be deleted?") → flag in summary, do not block writes.

---

## Notes

- `glab` only, not `gh`. Repo remotes under GitLab host.
- Don't review code quality. Don't flag bugs. Documentation only.
- Anti-patterns rejected during review → capture under "Gotchas".
- Respect existing CLAUDE.md tone and structure of each repo.
- Skip writes to files outside workspace root.
