# claude-workflow

Personal [Claude Code](https://claude.com/claude-code) slash commands for a plan-driven dev workflow at [Flat](https://flat.io) (Vue 3 / TypeScript / pnpm, GitLab + Linear + Notion).

The flow is **plan → investigate → execute → review**, with plans living as markdown files in `CLAUDE_PLANS/` and tracked on a Notion dev board.

## Install

Symlink (or copy) the commands into your Claude Code commands dir:

```bash
git clone git@github.com:bezdonas/claude-workflow.git
ln -s "$PWD/claude-workflow/commands"/*.md ~/.claude/commands/
```

Then invoke any command in Claude Code with `/<name>`.

## Commands

### Planning

| Command | What it does |
|---------|--------------|
| `/make-plan <Linear ID/URL or description>` | Create a fully-developed plan file in `CLAUDE_PLANS/` from a Linear issue or text description. Stops before any code or git side effects. |
| `/pre-investigate-plan <plan.md>` | Resolve open questions, confirm code paths, fill blocking unknowns before implementation. |
| `/criticize-plan <plan.md>` | Critique a plan — surface gaps, wrong assumptions, missing edge cases, weak decisions. |
| `/explain-plan <plan.md>` | Generate a human-readable HTML explainer for a plan. Architecture / causation / module-map focus, not code listing. |

### Execution

| Command | What it does |
|---------|--------------|
| `/work-plan [plan.md]` | Execute a plan's Next-session-focus. Resume-style — one focused chunk per invocation, updates the plan in place. Never pushes without explicit confirmation. Defaults to most-recently-modified plan. |
| `/pre-clear-handoff <plan.md>` | Pre-`/clear` handoff — record everything valuable from current context into an md file so a fresh session picks up cleanly. |

### Review

| Command | What it does |
|---------|--------------|
| `/local-review [MR/paths]` | Code review local branch (vs `develop`) or a GitLab MR. Never posts. Runs on drafts. |
| `/check-e2e <MR-url-or-pipeline-id>` | Compare MR pipeline e2e failures vs the `develop` baseline. |

### Knowledge / housekeeping

| Command | What it does |
|---------|--------------|
| `/mine-project <Linear project>` | Mine a finished Linear project's MRs for reusable patterns and write them into relevant `CLAUDE.md` files. |
| `/sync-board [--dry-run]` | Sync the Notion Dev Task Board — compute each Linear task's stage from ground truth, run auto-stage actions (`make-plan` / `local-review` + `check-e2e`), flag rows needing attention. Never commits or pushes. |

## Conventions

- **GitLab** via `glab` CLI; branch from `develop`, MR back to `develop`.
- **Plans** are the source of truth — markdown in `CLAUDE_PLANS/`, one focused chunk of work per `/work-plan` run.
- Review commands are **read-only by default** — they never post or push without explicit confirmation.
