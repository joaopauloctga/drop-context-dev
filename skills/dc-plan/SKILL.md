---
name: dc-plan
description: >-
  Plan a Drupal feature grounded in Drop Context: runs the
  drop-context-developer agent in a forked context to consult installed dc-*
  skills and the drop-context MCP docs for the installed releases, and writes a
  Context Brief (facts with sources, decisions, steps, verification commands)
  to .drop-context/briefs/<slug>.md. Use when asked to plan, scope, or design a
  Drupal feature or change before implementing it. Heavy doc reading happens in
  the fork — only the brief path and a short summary come back.
context: fork
agent: drop-context-developer
argument-hint: <feature to plan>
---

# /dc-plan — produce a Context Brief

You are planning, not implementing: **do not create or edit any project code
or config in this run.** Your only write is the brief file (and its directory).

Feature to plan: use the invocation arguments; if they are empty or ambiguous,
state the interpretation you chose at the top of the brief instead of stopping.

## Procedure

1. **Map** — project root, command wrapper (ddev/lando/vendor), installed core
   and module versions for everything the feature touches
   (`drop-context project:status --json`, else composer.lock).
2. **Consult** — walk the source ladder (skills → MCP module docs → MCP core →
   pretraining-with-validation noted as such). Respect the budget: per module,
   `get_module` + `ai-integration` + at most 2 targeted docs. Collect every
   fact you will rely on **with its source**.
3. **Decide** — integration shape (config-only vs plugin vs subscriber vs
   service decoration…), what lives in which custom module, config schema,
   permissions. Prefer the shapes the docs recommend (`ai-integration` /
   `extension-points`).
4. **Write the brief** to `<project>/.drop-context/briefs/<slug>.md` (create
   the directory; slug = kebab-case of the feature). Use exactly these
   sections:

   ```markdown
   # Brief: <feature>
   ## Scope
   ## Installed stack
   ## Modules × catalog
   ## Sources consulted
   ## Facts
   ## Decisions
   ## Steps
   ## Verification
   ## Source reads
   ## Doc gaps
   ## Open questions
   ```

   - `## Facts`: one per line, `fact → source` — e.g.
     ``service `flag` (`FlagServiceInterface`) → dc-flag/references/use.md``.
   - `## Steps`: numbered, each naming its files and citing the facts it uses.
   - `## Verification`: concrete commands with the detected wrapper
     (`ddev drush cr`, `ddev drush php:eval …`, phpunit if present).
   - `## Source reads` / `## Doc gaps`: empty (`none`) unless the ladder's
     last rung was genuinely needed — then record case (i/ii/iii) and details.
5. **Return** — final message only: the brief path, then a summary of at most
   10 lines (scope, chosen shape, step count, open questions). No file dumps.
