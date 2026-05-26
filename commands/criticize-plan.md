---
allowed-tools: Read, Glob, Grep, Bash(git log:*), Bash(git show:*), Bash(ls:*), Bash(find:*)
description: Critique a plan md file — surface gaps, wrong assumptions, missing edge cases, weak decisions
argument-hint: <path to plan md file>
---

# Plan Critic

Target plan file: `$ARGUMENTS`

If empty, ask user for path. If file doesn't exist, stop and report.

## Goal

Be the sharpest reviewer that has ever read this plan. Look for what the author missed, not what they got right. Surface every assumption that is not proven. Find decisions that look defensible but break under realistic edge cases. Push back on hand-waving.

This is **investigation + critique only**. Do not modify the plan file. Do not write code.

## Workflow

### 1. Read the plan in full

Read `$ARGUMENTS` end to end. Read it twice if dense — once to absorb, once to map the structure.

### 2. Verify factual claims against the actual codebase

Every concrete claim in the plan (file path, function name, exported symbol, schema field, integration point, "X already does Y", "Z is already wired into the W pipeline") is a hypothesis to verify. Verify with `Read` / `Grep` / `Glob`. If the plan says "X lives at path/Y.ts:42", confirm it. If the plan references prior commits, `git log` / `git show`.

Mark each claim:
- ✅ verified
- ⚠️ partially correct (with what's actually true)
- ❌ wrong (with what's actually there)
- ❔ unverifiable from code alone (flag for the human)

### 3. Hunt for missing concerns

Walk through each architectural layer the plan touches. For each, ask:

- What does this layer **normally** integrate with that the plan didn't mention?
- What happens at the boundaries (entity lifecycle: create / read / update / delete / undo / redo / serialize / deserialize / migrate)?
- What other features touch the same data path and could break?
- What cleanup / orphan-state risks exist (UUID maps, caches, indexes, parallel data structures)?
- What error states are not addressed?
- What happens at edge values (empty, max, zero, overlap, simultaneous)?
- What concurrent edits / race conditions are possible?
- What does the plan implicitly assume about the user's input shape that may not hold?

### 4. Challenge decisions, not just gaps

For each major decision called out in the plan:
- Steelman the **alternative** that wasn't picked. Is the rejection well-reasoned, or based on a guess?
- Find the implicit assumption underneath the decision. Is that assumption load-bearing? Brittle?
- Look for the decision the plan doesn't realize it's making (silent default).

### 5. Sequencing + ordering critique

- Are commits actually independently green, or do they assume something not yet shipped?
- Does the cross-package / cross-MR ordering hold under "what if MR A reviewer asks for changes"?
- What is the minimum-blast-radius rollback story if something lands and turns out wrong?
- Are tests structured to fail loudly when the contract breaks, or just to pass when it works?

### 6. Forward-compatibility / future-tax

- Does the plan paint into a corner that the next feature in the same area will pay for?
- Is the schema choice truly extensible, or just extensible-for-this-case?
- Does the code style match the surrounding package's conventions (check the package's CLAUDE.md if present)?

## Output

Single response, ≤ 800 words. Structured:

```
## Verification of plan claims
- ✅ / ⚠️ / ❌ / ❔ list — only the items that matter (skip trivial ✅)

## Gaps I'd push back on
- Numbered, ranked by severity. Each: what's missing, why it bites, suggested fix shape (not full code).

## Decisions worth re-litigating
- For each: the implicit assumption, the alternative the plan dismissed, when the dismissal would be wrong.

## Edge cases not addressed
- Concrete scenarios. Each: trigger condition, expected behavior, plan's current behavior.

## What looks fine
- Brief — patterns the plan got right and should not be over-engineered further.

## Open questions for the human
- ❔ items + anything requiring product / design judgment, not code.
```

## Rules

- Be specific, not abstract. "Plan doesn't handle X" needs a concrete X.
- Cite file paths and line numbers when claiming the plan is wrong about the codebase.
- Do not be diplomatic about weak reasoning. State plainly which assumption is shaky and why.
- Do not invent issues. If the plan looks solid in a section, say so and move on.
- No code changes. No plan file edits. Investigation + critique only.
