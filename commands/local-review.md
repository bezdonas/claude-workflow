---
allowed-tools: Bash(glab mr view:*), Bash(glab mr list:*), Bash(glab mr diff:*), Bash(git diff:*), Bash(git log:*), Bash(git blame:*), Bash(git status:*), Bash(git rev-parse:*), Bash(git branch:*), Bash(git merge-base:*)
description: Code review local branch (vs develop) or a GitLab MR. Never posts. Runs on drafts.
argument-hint: MR ID/URL, file paths, or empty for current branch vs develop
---

# Local Code Review

Comprehensive code review for local changes or a GitLab merge request. **Terminal output only — never posts to GitLab. Does not skip drafts.**

## Arguments

Parse `$ARGUMENTS`:

- Contains `!<digits>` or `merge_requests` URL → **MR mode**
- File paths only → **Local mode, scoped**
- Empty → **Local mode**, default diff = current branch vs `develop`

**Flags:**

- `--verbose` — show all issues including filtered ones
- `--no-security` — skip security audit
- `--base <ref>` — override develop as the comparison base (Local mode only)

---

## Workflow

### Step 0: TodoWrite

Build a todo list tracking each step.

### Step 1: Mode detection + diff acquisition

**MR mode:**

- Extract MR ID from argument
- `glab mr view <ID>` for metadata (title, source/target branch, author)
- `glab mr diff <ID>` for the diff
- **Skip draft / WIP / closed gating.** Review them regardless.

**Local mode:**

- Default base ref = `develop` (or `--base <ref>` if given)
- Resolve base via `git merge-base HEAD <base>` → `<merge-base-sha>`
- Diff: `git diff <merge-base-sha>...HEAD` (three-dot — diff vs branch point, not vs current `develop` tip)
- If file paths provided: scope diff to those paths

If the diff is empty, stop with "No changes to review."

### Step 2: Context gathering

Launch 2 **Haiku agents** in parallel:

**Agent A — CLAUDE.md discovery:**

- Find root `CLAUDE.md`
- Find `CLAUDE.md` in every directory whose contents the diff modifies
- Return list of paths

**Agent B — Change summary:**

- Read the diff
- Return short summary: files modified, nature of changes (new feature / refactor / fix / test-only / formatting-only), packages touched

### Step 3: Parallel review

Launch **5 Sonnet agents in parallel** (skip Agent 5 in Local mode unless a remote branch exists; conditionally launch Agent 6).

Each agent returns a list of issues with: description, `file:line`, reason category, suggested fix shape (if applicable).

**Agent 1 — Bug detection:**

Shallow scan for obvious bugs in the diff:

- Logic errors and edge cases
- Null / undefined handling
- Race conditions
- Off-by-one errors
- Obvious security holes

Focus on real bugs. Skip nitpicks and likely false positives. Avoid reading far beyond the changes.

**Agent 2 — CLAUDE.md compliance:**

Read every `CLAUDE.md` from Step 2 Agent A. Flag diff lines that violate explicit rules. Only flag what `CLAUDE.md` explicitly calls out — not implicit style preferences.

**Agent 3 — Code quality:**

- Readability + simplicity
- DRY violations (real duplication, not coincidental similarity)
- Missing error handling on critical paths
- Naming clarity
- Structural problems (god-files, misplaced concerns)

Significant issues only. No nitpicks.

**Agent 4 — Git history:**

For each modified file, run `git blame` and `git log` on the surrounding lines. Identify:

- Patterns that have caused bugs before in this file
- Frequently-rewritten regions (high-churn = risk)
- Context from previous commits that informs the current change

**Agent 5 — Previous MR scan (MR mode, or Local mode with remote tracking):**

- `glab mr list` — find previous MRs touching the same files
- Read comments on those MRs for feedback that may apply again
- For pure Local mode with no remote: skip or use `git log` of touched files

**Agent 6 — Security audit (conditional, unless `--no-security`):**

Launch if diff touches auth / user input / DB queries / API endpoints / session handling / secrets / serialization:

- Input validation + sanitization (beyond schema-typed inputs — business rules)
- NoSQL injection on raw queries / aggregations
- Authorization checks
- XSS
- Secrets handling (no hardcoded creds, env var usage)

### Step 4: Confidence scoring

For each issue found in Step 3, launch a parallel **Haiku agent** to score confidence. Each agent gets:

- Issue description
- Relevant code context
- List of CLAUDE.md paths

**Rubric (give verbatim to scorer):**

Score 0-100:

- **0-25** — False positive / uncertain. Pre-existing issue. Stylistic without CLAUDE.md backing.
- **50** — Real but nitpicky / unlikely in practice.
- **75** — Verified real issue that will likely be hit, or explicit CLAUDE.md violation.
- **100** — Critical. Definitely a bug or violation, evidence directly confirms.

For CLAUDE.md flags, double-check the CLAUDE.md actually calls the issue out.

### Step 5: Filter

Drop issues scored < 75 (unless `--verbose`).

Filter-out examples:

- Pre-existing issues not introduced by the diff
- Looks-like-bug but is not
- Pedantic nitpicks
- Things a linter / typechecker / compiler catches (missing imports, type errors, formatting)
- General quality opinions not in CLAUDE.md
- Issues silenced in code (lint-ignore comments)
- Intentional functionality changes
- Real issues on lines NOT in the diff

If nothing passes the filter, output "No issues found."

### Step 6: Output (terminal only)

```markdown
## Code Review

Mode: <MR <ID> | Local vs <base-ref>>
Reviewed: <X> files, <Y> lines changed
Found: <N> issues (filtered <M> low-confidence)

### Critical issues

1. **<Issue title>** (confidence: <score>)
   `<file:line>`
   <Description>

   Suggested fix:
   ```
   <code suggestion>
   ```

### Important issues

2. **<Issue title>** (confidence: <score>)
   `<file:line>`
   <Description>

   Reference: `<CLAUDE.md path>:<line>` (if applicable)
```

If `--verbose`, append a "Filtered (low confidence)" section listing skipped issues with their scores.

---

## Hard rules

- **NEVER post.** No `glab mr note`. No `glab mr comment`. Terminal output only.
- **NEVER skip drafts.** Always run the review regardless of MR status.
- Use `glab` (not `gh`) for any GitLab read operation.
- For Local mode, default comparison base is `develop` via `git merge-base`. Override with `--base`.
- No build / typecheck signal — CI runs those.
- Each flagged issue must cite `file:line` and either a code snippet or CLAUDE.md reference.
- TodoWrite first, update as you progress.
- Provide ≥ 1 line of context before and after when referencing code.
