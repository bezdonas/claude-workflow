---
allowed-tools: Read, Glob, Grep, Bash(git log:*), Bash(git show:*), Bash(git diff:*), Bash(ls:*), Bash(find:*), WebFetch
description: Pre-investigate a plan md file — resolve open questions, confirm code paths, fill blocking unknowns before implementation
argument-hint: <path to plan md file>
---

# Plan Pre-Investigator

Target plan file: `$ARGUMENTS`

If empty, ask user for path. If file doesn't exist, stop and report.

## Goal

Reduce uncertainty in the plan before any implementation work starts. Resolve open questions, confirm assumptions, surface blocking unknowns, find the actual code shape the plan needs to integrate with. **Investigation only — no code changes, no plan rewrites.** Output is a structured report the human (or a future implementation session) can use to update the plan or unblock work.

A good pre-investigation pass turns "we'll figure that out during implementation" into "here is the exact file / function / symbol you need, here is the existing pattern to mirror, here is the gotcha to avoid."

## Workflow

### 1. Read the plan in full

Read `$ARGUMENTS` end to end. Note every:
- Section labeled "Open questions", "Blocking unknowns", "TBD", "investigate during ...", "Risks", "Plan divergence", or similar
- Concrete file path, function name, or exported symbol the plan references
- Decision marked as "if X turns out hard, fall back to Y"
- Claim about how an existing system works ("X already does Y", "Z lives at W")
- Cross-package / cross-MR dependency

### 2. Build an investigation list

Enumerate every item that needs verification. Prioritize:

1. **Blocking unknowns** — anything that gates the plan
2. **Load-bearing claims** — anything downstream decisions depend on
3. **Risk callouts** — anything the plan author flagged as risky
4. **Fallback decisions** — fork points where the plan branches based on what's discovered
5. **Integration shapes** — the exact API / type / interface the plan needs to plug into

Use a TodoWrite-style mental list. Work top-down by priority.

### 3. Verify each item

For each investigation target, use the tools to find ground truth:

- **File / symbol exists?** → `Glob` / `Grep` / `Read`
- **Function signature / return shape?** → `Read` the source
- **Caller graph (who depends on this)?** → `Grep` for callers
- **History (when did this change, what shape did it have before)?** → `git log` / `git show`
- **Similar pattern already in codebase?** → `Grep` for the analog
- **External documentation needed?** → `WebFetch` if a URL is given in the plan

Capture the answer in concrete form: paths, line numbers, function signatures, code snippets where the shape is non-obvious.

### 4. Resolve fallback decisions

For each "if X is hard, fall back to Y" fork in the plan, investigate enough to know which branch is the one to take. State the call with reasoning.

### 5. Discover what the plan didn't think to ask

Once you understand the area the plan touches, look for:

- Adjacent code paths that will need to change but the plan didn't mention
- Cleanup / lifecycle hooks the plan must wire into (e.g. measure removal, dispose, destroy, GC)
- Naming / convention requirements set by package-local CLAUDE.md files
- Tests / fixtures the plan must extend
- Snapshots / generated artifacts the plan must regenerate

Surface these as **additional findings**, not blockers — let the human decide whether to fold them into the plan.

## Output

Single response, ≤ 800 words. Structured:

```
## Plan summary (1-2 sentences)
What the plan is trying to build, restated.

## Blocking unknowns — resolved
For each item the plan marked as blocking:
- Question
- Answer (with file:line citations)
- What this means for the plan

## Blocking unknowns — still open
For each item that could not be resolved without human input:
- Question
- Why code-level investigation alone is insufficient
- What the human needs to decide

## Load-bearing claims — verification
Plan claims about existing code, marked ✅ / ⚠️ / ❌, with citations.

## Fallback decisions — call made
For each fork in the plan:
- Branch picked, with reasoning
- File evidence supporting the call

## Additional findings (plan should incorporate)
Things the plan didn't think to address but should:
- Adjacent code paths
- Cleanup hooks
- Convention requirements
- Tests / fixtures / snapshots

## Recommended plan edits
Concrete bullet list of edits the plan file should receive (do NOT apply them — list only).
```

## Rules

- Investigation only. Do not modify the plan file. Do not write code.
- Every concrete claim in your report must be backed by a file path + line number or a quoted snippet.
- Where the answer requires human judgment (product decision, design call, deadline), say so and stop — don't guess.
- If the plan is already well-investigated and you find nothing new, say so honestly and keep the report short.
- Respect the plan's stated scope. Don't expand the feature.
