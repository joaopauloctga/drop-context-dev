---
name: dc-consult
description: >-
  The Drop Context source ladder — how to ground any Drupal task in installed
  dc-* skills and the drop-context MCP docs instead of reading contrib/core
  source or trusting pretraining. Load before touching Drupal code when the
  drop-context MCP is available: it tells you which source to consult in which
  order, when reading module/core source is legitimate, and what to record.
  For planning a whole feature use /dc-plan; for executing one use
  /dc-implement.
---

# dc-consult — the Drop Context source ladder

You are working in a Drupal project that has the **drop-context** MCP server
(`list_modules`, `get_module`, `get_doc`, `list_core_libraries`,
`get_core_library`, `get_core_library_doc`) and the `drop-context` CLI. Facts
about modules come from there — not from reading contrib/core source, and not
from pretraining (which may describe another release).

For a multi-step feature, don't follow this inline — run **`/dc-plan
<feature>`** (consults in a forked context, writes a Context Brief) and then
**`/dc-implement <brief>`**. Use this skill directly for small tasks and
one-off questions.

## The ladder (always in this order)

1. **Map the target + installed versions** — `drop-context project:status
   --json` if available; else composer.lock / `.info.yml` /
   `Drupal::VERSION`. Never assume a version.
2. **Installed skill** — `<project>/.claude/skills/dc-<module>/` (machine name,
   underscores → hyphens): read SKILL.md, then only the `references/*.md` the
   task needs. Beats the MCP for the same module.
3. **Available skill** — `drop-context skill:list --search <module> --json`;
   install with `skill:add <name> --json --force` when the module is central.
4. **MCP module docs** — `get_module(<machine>)` first (docs index + releases),
   then `get_doc(<machine>, ai-integration, release=<installed or closest>)`,
   then at most 2 targeted docs. One doc ≈ 2–4k tokens; never fetch all.
5. **MCP core** — core modules (node, views, field, …) via the same flow;
   framework libraries (Ajax, Batch, Queue, Hook, Flood) via
   `get_core_library_doc(<lib>, usage)`.
6. **Pretraining** — only for core APIs absent from the catalog, and validated
   with `drush php:eval` before use. Never for contrib.
7. **Source — last, and only for one of three reasons**:
   (i) the module isn't in the catalog at all; (ii) the specific fact is absent
   from the right docs; (iii) **you followed the doc exactly and it didn't
   work** — the doc may be wrong; the source is your check. Record the reason
   (and, for iii, the doc×source contradiction) before reading, then read
   surgically — grep one symbol, never "open the module".

The project's own code (`web/modules/custom/**`, themes, config, tests) is
never restricted — the ladder governs `web/modules/contrib/**`, `web/core/**`,
`vendor/**` only.

## Version rule

Match the docs release to the **installed** version (`available_releases` in
`get_module`; unknown release = hard error). Same line → use + note delta;
different line → orientation only, verify load-bearing facts on the site.

## Always end with a grounding report

List: skills used, MCP calls made, pretraining facts (and how validated),
source reads (with their case i/ii/iii), doc gaps found. No API identifier
(service, hook, event, route, permission, config key, plugin ID) without a
source or an on-site verification.
