---
description: Sync Notion Dev Task Board — compute each Linear task's stage from ground truth, run auto-stage actions (make-plan / local-review + check-e2e), flag rows needing attention. Never commits, never pushes.
argument-hint: optional --dry-run (compute + print, no actions, no Notion writes)
---

# Task Board Orchestrator

Reads the Notion **Dev Task Board**, computes each task's stage from ground truth, advances auto-stages, writes the projection back. Designed to run unattended under `/loop`.

## Inviolable safety contract

1. **Allowlist of side-effecting actions:** `make-plan`, `local-review`, `check-e2e`, `glab` (read-only subcommands only), Notion writes, Linear reads. NOTHING else.
2. **NEVER** invoke `work-plan`. **NEVER** `git commit / branch / checkout / push`. **NEVER** `glab` mutations (`mr create`, `mr note`, `mr update`, resolve threads). `glab` is read-only here (`mr view`, `mr list`, discussions read).
3. If any sub-skill would hit an interactive prompt (AskUserQuestion), **skip that row, set no stage, append it to the "needs manual attention" report**, and continue. Never answer a prompt by guessing.
4. `--dry-run`: compute and print the stage table only. No sub-skill calls, no Notion writes, no glab calls beyond read.

## Canonical locations

- Plans + review files: `~/devprojects/tutteo/CLAUDE_PLANS/` (always; never another checkout).
- Notion DB: find by title `Dev Task Board` via `notion-search` (no hardcoded id).

## Procedure (one cycle)

### 1. Load board

- `notion-search` → resolve `Dev Task Board` database id.
- `notion-fetch` → all rows. For each row read: `Linear ID`, `Stage`, `Plan file`, `MR`.

### 2. Per-row stage computation

For each row, **in this precedence order**:

**2.0 Terminal guard** — if current `Stage` ∈ {`Done`, `Archived`} → **skip entirely** (do not read ground truth, do not write).

**2.1 Resolve plan file**
- If `Plan file` set → check the file exists in canon dir.
  - If field set but file MISSING → **anomaly**: do NOT re-plan. Leave Stage unchanged, add row to attention report with note "plan file recorded but missing", continue to next row.
- If `Plan file` empty → resolve in canon dir, in this order (stop at first hit). A plan that merely *mentions* another ticket's ID in its body must NOT match — only a dedicated plan counts:
    1. **Filename** contains the ID: `ls ~/devprojects/tutteo/CLAUDE_PLANS/ | grep -i "<LINEAR_ID>"`.
    2. ID appears in the plan's **H1 heading** or **Status / Per-ticket progress** block. Grep for the ID, then for each hit open the file and confirm the ID sits in the H1 or the `Per-ticket progress` table — ignore body-only mentions.
  - Exactly one confirmed file → backfill `Plan file` (writeback, see step 3) and treat as "plan exists".
  - More than one confirmed → **ambiguity anomaly**: do not pick, do not re-plan; flag row to attention report.
  - None confirmed → **plan does not exist**.
- Note: Notion may auto-linkify a `Plan file` value containing `.md` as `NAME[.md](http://NAME.md)`. Strip the markdown-link wrapping and use the bare filename token.

**2.2 No plan → stage `Planning` (AUTO)**
- (unless `--dry-run`) invoke `make-plan <LINEAR_ID>` — it writes only a plan file to canon dir, no git.
  - Precondition guaranteed here: plan does not exist (no overwrite prompt), explicit ID (no "what to plan" prompt), fixed cwd = this session (no folder prompt). If make-plan still prompts → skip per safety rule 3.
- After make-plan: set `Plan file` to the new file path, `Stage` = `Planning` → next cycle re-evaluates (now has plan, → Plan polish).
- Stop processing this row this cycle.

**2.3 Plan exists, resolve MR via Linear**
- `get_issue <LINEAR_ID>` → scan attachments for a GitLab MR URL.
- If no MR attachment → **stage `Plan polish`** (hand-off, 🧍 — Ramil preinvestigates/grills/builds). Write stage. Done with row.
- If MR present → record MR URL (writeback `MR` field), extract MR iid + branch.

A ticket may have **several MRs** (e.g. a `flat` MR + a `music` MR). The ticket's MR set = all GitLab MR attachments from Linear; `<repo>` is parsed from each MR URL.

**2.3b All MRs merged → stage `Done` (AUTO, terminal)**
- For each MR: `glab mr view <iid> -R <repo>` → `state`.
- If the MR set is non-empty AND every MR `state == merged` → **stage `Done`**. This is the ONE case where the orchestrator itself writes `Done`. Done with row.
- If some merged + some open → continue below, treating the open MRs normally.

**2.4 Plan + MR: external threads first (#7 override)**
- For EACH MR in the set: `glab mr view <iid> -R <repo>` + discussions read → count UNRESOLVED threads authored by someone other than Ramil (`ramil@tutteo.com` / his GitLab handle).
- Sum across MRs. If total > 0 → **stage `External review feedback`**, write `Ext threads` = total. OVERRIDES Ready-to-test/Done-adjacent states. Done with row.
- Else `Ext threads` = 0.

**2.5 Review freshness**
- Review file = `~/devprojects/tutteo/CLAUDE_PLANS/<LINEAR_ID>_REVIEW.md` — keyed by **ticket ID**, NOT plan TOPIC (one plan may cover several tickets; one ticket may have several MRs).
- For EACH MR: read head SHA `glab mr view <iid> -R <repo>` (diff_refs.head_sha).
- Read the per-MR `Reviewed-SHA` lines from the review file header if it exists.
- **Frozen-stage guard:** if current Notion `Stage` ∈ {`Ready to test`, `Code review`} → do NOT auto re-review and do NOT change the stage. These are user-owned: `Ready to test` = Ramil is testing; `Code review` = Ramil un-drafted the MR and it awaits a human reviewer (he sets this stage manually, after testing). Only steps 2.3b (→ `Done` on full merge) and 2.4 (→ `External review feedback` on new external threads) above may move a row off these. Done with row.
- STALE if: file missing, OR any MR absent from the header, OR any MR's head SHA != its recorded `Reviewed-SHA`.

**2.6 Stale review → stage `Claude review` (AUTO)**
- (unless `--dry-run`) for EACH MR run `local-review` (read-only; never posts) + `check-e2e` for that MR's pipeline.
- Write `~/devprojects/tutteo/CLAUDE_PLANS/<LINEAR_ID>_REVIEW.md`:
  ```
  # <LINEAR_ID> — Local Review
  Date: <today>
  MRs:
    - <repo>!<iid> @ <head sha> — check-e2e: <pass | N new failures vs develop>
    - ...
  Findings: <N total>
  ```
  followed by per-MR local-review findings + check-e2e summary.
- Set `Review findings` = N total. Set `Stage` = `Claude review` → next cycle classifies by N.
- Done with row.

**2.7 Fresh review → classify by findings**
- Read `Findings: N` from review header.
- N > 0 → **stage `Claude review findings`** (🧍 — Ramil fixes via fixups; push → SHA changes → re-review next cycle).
- N == 0 → **stage `Ready to test`** (🧍 — Ramil tests by hand, then manually sets `Code review` once he un-drafts the MR for human review).
- Write stage + `Review findings`.

### 3. Notion writeback rules

- Write computed `Stage` for every row EXCEPT terminal (skipped at 2.0).
- Always update `Last synced` = today for processed rows.
- Update `Plan file`, `MR`, `Review findings`, `Ext threads` when newly derived.
- Never write `Archived`, and never overwrite a Ramil-set `Done`. The orchestrator writes `Done` itself ONLY via step 2.3b (all of a ticket's MRs merged).

### 4. Report

Print:
```
## Board sync — <date>
Auto-actions this cycle: <make-plan: [IDs]> <reviewed: [IDs]>
Needs your attention:
  - <ID> — <stage> — <one-line why>
Anomalies/skipped: <list with reason>
```

## Notes
- One cycle = full board pass. `/loop <interval> /sync-board` to repeat.
- Cost: each `Claude review` row runs local-review + check-e2e. Acceptable per design.
- Manual stages Ramil owns: `Code review` (set when MR un-drafted, awaiting human review). `Done`/`Archived` (also settable manually). Orchestrator never sets `Code review` itself; it only moves rows off it via merge (→ Done) or new external threads (→ External review feedback).
