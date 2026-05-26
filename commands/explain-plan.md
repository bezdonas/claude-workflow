---
description: Generate a human-readable HTML explainer for a plan md file. Architecture/causation/module-map focus, not code listing.
argument-hint: path to a plan md file (e.g. CLAUDE_PLANS/A_TEMPO.md)
---

# Plan Explainer

Input: `$ARGUMENTS` — path to a plan md file.

## Goal

Produce a **human-readable HTML explainer** at `<plan-path>.explained.html` (same directory as the input plan). The HTML targets a developer who already knows the codebase but wants the *mental model* in 2 minutes, not the full engineering spec.

This command:

- **Writes one HTML file.** No code edits. No git side effects.
- **Does not duplicate the plan.** It interprets and visualizes it.
- **Does not list files line-by-line.** That's what the plan md is for.
- **Focuses on architecture, causation, repo/module map, invariants.**

## Workflow

### 1. Parse arguments

`$ARGUMENTS` must resolve to an existing md file. If empty or file missing, **stop and ask user** for the path. Do not guess a "similar" plan.

### 2. Read the plan

Read the full plan md. Identify:

- **Feature summary** — what + why
- **Touched repos / packages** — the cross-repo footprint
- **Phases / sequencing** — but read for *causation*, not for the file list
- **Cross-cutting invariants / checklist** — non-obvious rules
- **Blocking unknowns** (`❔`) — open decisions
- **MR / branch plan** — high-level shape only (which repo gets a branch, why split)

### 3. Read minimal context (if needed)

If the plan references package names you don't recognize (e.g. `adagio-display`, `domToretto`, `music`), do a *one-line* probe — read the package's `CLAUDE.md` first paragraph or `package.json` description. Goal: know what each module *is* for the diagram. Do not deep-explore the code.

### 4. Compose the HTML

Single self-contained HTML file. Use the template below. Embed mermaid via CDN (`https://cdn.jsdelivr.net/npm/mermaid@10/dist/mermaid.min.js`).

Sections (in order):

1. **TL;DR** — 2-3 sentences. What this feature does and the single most important architectural decision.
2. **Repo / package map** — mermaid `graph LR` diagram. Nodes = touched packages, grouped by repo. Edges = dependency direction. Annotate which package owns which layer (engine, display, UI).
3. **Causation flow** — mermaid `sequenceDiagram` or `flowchart TD`. User action → engine action → display refresh → UI feedback. Show what triggers what across layers. This is the most important diagram.
4. **Layer responsibilities** — short table. Layer | Owns | Boundary rule. E.g. "engine owns mutation; display owns rendering; UI owns intent + gating".
5. **Invariants / non-obvious rules** — bullet list. Each rule has *what* + *why*. Pull from plan's cross-cutting checklist AND from memory-style gotchas embedded in the plan.
6. **Open questions** — each `❔` from Blocking unknowns rendered as a card: question, options, recommendation, what would decide it.
7. **MR shape** — short paragraph per repo: "one branch in `<repo>`, N commits, MR title shape". No commit-by-commit listing.

### 5. Tone + style rules

- **Plain prose**, not engineer-spec bullets. Imagine explaining to a teammate at a whiteboard.
- **Diagrams over walls of text.** If a relationship can be a 5-node graph, draw it.
- **Name things in domain terms**, not in file paths. "Tempo marking" not `TempoDirectionType`.
- **Explain *why*, not *what*.** "We gate at UI because engine actions must stay free for cleanup paths" beats "premiumFeature lives on the public toggle action".
- **No code blocks** unless quoting a 1-line invariant. The plan has the code references; this file is the map.
- **No emojis** other than the `❔` carried over from the plan, unless user explicitly asked.

### 6. HTML template

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="utf-8">
<title><PLAN TITLE> — Explainer</title>
<style>
  :root { color-scheme: light dark; }
  body { font: 16px/1.55 -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif;
         max-width: 880px; margin: 2rem auto; padding: 0 1.25rem;
         background: #fafafa; color: #1a1a1a; }
  @media (prefers-color-scheme: dark) {
    body { background: #1a1a1a; color: #e8e8e8; }
    .card { background: #242424; border-color: #333; }
    code { background: #2a2a2a; }
  }
  h1 { margin-bottom: 0.25rem; }
  .sub { color: #888; font-size: 0.9rem; margin-bottom: 2rem; }
  h2 { margin-top: 2.5rem; padding-top: 1rem; border-top: 1px solid #ddd; }
  .card { border: 1px solid #e2e2e2; border-radius: 8px; padding: 1rem 1.25rem;
          margin: 1rem 0; background: #fff; }
  .card h3 { margin-top: 0; }
  table { border-collapse: collapse; width: 100%; margin: 1rem 0; }
  th, td { text-align: left; padding: 0.5rem 0.75rem; border-bottom: 1px solid #e2e2e2; vertical-align: top; }
  th { background: rgba(0,0,0,0.04); font-weight: 600; }
  code { background: #f0f0f0; padding: 0.1em 0.35em; border-radius: 3px; font-size: 0.9em; }
  .mermaid { background: #fff; padding: 1rem; border-radius: 8px; margin: 1rem 0; }
  @media (prefers-color-scheme: dark) { .mermaid { background: #fff; } }
  .tldr { font-size: 1.1rem; line-height: 1.6; }
  .question { border-left: 3px solid #d97706; padding-left: 1rem; }
  .meta { color: #888; font-size: 0.85rem; }
</style>
<script src="https://cdn.jsdelivr.net/npm/mermaid@10/dist/mermaid.min.js"></script>
<script>mermaid.initialize({ startOnLoad: true, theme: 'default' });</script>
</head>
<body>

<h1><PLAN TITLE></h1>
<div class="sub">Source: <code><PLAN PATH></code> · Generated <DATE></div>

<h2>TL;DR</h2>
<p class="tldr"><2-3 sentences></p>

<h2>Repo / package map</h2>
<p><One-paragraph orientation — which repos, who owns what.></p>
<div class="mermaid">
graph LR
  subgraph repo_X[<repo name>]
    pkgA[<package> — <layer>]
    pkgB[<package> — <layer>]
  end
  subgraph repo_Y[<repo name>]
    pkgC[<package> — <layer>]
  end
  pkgA --> pkgC
  pkgB --> pkgC
</div>

<h2>How it flows end-to-end</h2>
<p><Paragraph naming the trigger and the visible outcome.></p>
<div class="mermaid">
sequenceDiagram
  participant U as User
  participant UI as UI layer (<pkg>)
  participant E as Engine (<pkg>)
  participant D as Display (<pkg>)
  U->>UI: <action>
  UI->>E: <engine action>
  E->>E: <mutation>
  E->>D: <event>
  D->>U: <visible change>
</div>

<h2>Layer responsibilities</h2>
<table>
  <tr><th>Layer</th><th>Owns</th><th>Boundary rule</th></tr>
  <tr><td><layer></td><td><what it owns></td><td><what it must not do></td></tr>
</table>

<h2>Invariants you can't break</h2>
<div class="card">
  <h3><Invariant name></h3>
  <p><What + why in plain prose.></p>
</div>

<h2>Open questions</h2>
<div class="card question">
  <h3>❔ <Question></h3>
  <p><strong>Options:</strong> <A / B / C></p>
  <p><strong>Recommendation:</strong> <X — because ...></p>
  <p><strong>What would decide it:</strong> <evidence></p>
</div>

<h2>MR shape</h2>
<p><Per-repo paragraph: one branch, rough commit count, what each MR carries.></p>

<div class="meta">Generated by <code>/explain-plan</code>. Re-run to refresh.</div>

</body>
</html>
```

### 7. Write + report

Write to `<plan-path>.explained.html` (same directory, sibling file). If file exists, overwrite without asking — it's a regenerable artifact.

Print:

- Output path
- `open <path>` command for convenience
- Count of diagrams emitted, count of open questions surfaced

## Rules

- **No code edits.** Only the HTML file is written.
- **No git side effects.**
- **Single self-contained HTML.** No external assets except the mermaid CDN script.
- **If plan file missing or unreadable:** stop and ask. Do not substitute a "similar" plan.
- **If plan has no `❔` blocking unknowns:** omit the Open questions section entirely, don't fill with placeholders.
- **If plan is single-repo:** still emit the map diagram (nodes = packages within the repo), don't skip it.
- **Audience is a developer who knows the codebase.** Don't explain what Vue or pnpm are. Do explain what *this feature's* layer split is.
