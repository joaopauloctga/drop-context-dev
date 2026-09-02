---
name: dc-implement
description: >-
  Implement a Drupal feature from a Context Brief produced by /dc-plan (or a
  small feature directly): executes the brief's steps grounded in its
  fact-with-source list, consults Drop Context (skills/MCP) only for facts the
  brief lacks, verifies with drush/phpunit, and ends with a grounding report.
  Use when asked to implement, build, or execute a planned Drupal feature, or
  after /dc-plan finishes.
argument-hint: <brief slug or path — or a small feature description>
---

# /dc-implement — execute a Context Brief

1. **Load the protocol** — invoke the `dc-consult` skill (Skill tool) if its
   ladder isn't already in context. It governs every fact you use here.
2. **Resolve the brief** — argument may be a path, a slug under
   `<project>/.drop-context/briefs/`, or nothing (pick the most recent brief;
   say which). If the argument is a feature description and no brief exists:
   for a small task, proceed inline under the ladder; for anything multi-step,
   run the consult phase first and write the brief before coding.
3. **Implement from the brief** — the `## Facts` section is your fact base;
   do not re-fetch what it already answers. When a needed fact is missing,
   walk the ladder for that one fact and **append it to the brief** (keep the
   brief the single record). Follow the brief's `## Steps`; if reality forces
   a deviation, note it under `## Decisions`.
4. **Respect the boundaries** — never edit `web/modules/contrib/**`,
   `web/core/**`, `vendor/**`; hooks via `#[Hook]` attribute classes; config
   as YAML with proper `dependencies:`; services injected via interfaces.
5. **Verify** — run the brief's `## Verification` commands with the detected
   wrapper (`ddev drush` / `lando drush` / `vendor/bin/drush`): `drush cr`
   clean, `drush php:eval` smoke checks, project tests when present. A failing
   check is not done — fix or report it honestly. If the docs said X and the
   site does Y (a case-iii situation), verify against source surgically and
   record the contradiction under `## Doc gaps`.
6. **Close** — update the brief (`## Source reads`, `## Doc gaps`, new facts),
   then end your message with the mandatory grounding report:

   ```markdown
   ## Grounding report
   - Skills: …
   - MCP: …
   - Pretraining: … (how validated)
   - Source reads: none | <module> — case (i/ii/iii) — <path:line> — <why>
   - Doc gaps: none | <doc id @ release> — <what's missing/wrong>
   ```
