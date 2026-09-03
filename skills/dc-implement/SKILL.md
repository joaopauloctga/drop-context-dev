---
name: dc-implement
description: >-
  Implement a Drupal feature with the drop-context MCP as the source of truth:
  reads the project (composer.json / composer.lock, custom modules, config),
  matches the request against modules already installed and catalog modules
  worth installing, reads their versioned docs, then builds, installs what is
  needed, verifies with drush and the project's tests, and ends with a
  grounding report. Works directly from the request; when a plan from /dc-plan
  exists for it, the plan is the starting fact base. Use when asked to
  implement, build, add, or change something in a Drupal project.
argument-hint: <what needs to be done — or a plan slug/path from /dc-plan>
---

# /dc-implement — build it

You work in the main context so the user sees every change. The rules of the
`drop-context-developer` agent apply here verbatim — the short version:

- **Facts come from the project and the drop-context MCP** — never from
  contrib/core source (three exceptions: module not in the catalog; fact absent
  from the right docs; the doc was followed and did not work) and never from
  unvalidated pretraining.
- MCP parameter names: `list_modules({query})`,
  `get_module({module, release?})`, `get_doc({module, doc?, release?})` —
  `module` is the machine name; `get_core_library_doc({library, doc?})`.
  `doc` defaults to `ai-integration`; `release` must be a tag from
  `available_releases`, or omitted.
- Never edit `web/modules/contrib/**`, `web/core/**`, `vendor/**`.

## Rules

Before anything else, read `references/drupal-rules.md` (next to this file).
It governs every line you write: events before hooks; hooks verified through
the MCP and as specific as possible; no business logic in hooks; services in
`src/Services/` registered by class name and always injected; schema for every
config and the import → export → adopt cycle; complete field/display config;
services of other modules injected as optional (`@?`) so a deploy never
breaks on a module not yet enabled; no tests unless asked; drush only
against the local site; access and escaping; cache metadata studied per
case, not sprinkled; no queues or batches unless asked; comments within the
length limits of rule 10 (inline ≤ 2 lines, ≤ 4 for a business rule; method
docblock summary + ≤ 3 lines; one-line `@param`/`@return`; class docblocks
that show how to use the class; and never a comment that narrates the change
you are making — comments describe the code as it is); a drush script is a
local convenience only — what it did must become exported YAML or an update
hook (`post_update` before `cim`, `deploy` hook after), and when it is
unclear which one or where it lives, ask the user (rule 11); everything you
write — code, comments, UI strings, config labels, messages — is in English,
whatever language the request came in (rule 12).

## Input

The argument is either **the request itself** ("add a bookmark flag on
articles") or **a plan** — a path, or a slug under
`<project>/.drop-context/plans/`. A plan is a head start, not a requirement:
with one, its `## Modules`, `## Steps` and `## Docs consulted` are your
starting fact base and you skip what it already settled; without one, you do
the discovery yourself, right here. An empty argument is the one case to ask:
what should be built?

## Procedure

1. **Read the project**: core version and installed modules with versions
   (`composer.lock`), custom modules, the wrapper (`.ddev/` → `ddev drush` /
   `ddev composer`; `.lando.yml` → `lando …`; else `vendor/bin/drush`), and
   the existing config/code the request touches. With a plan, just confirm
   `composer.lock` still matches its `## Modules`.
2. **Discover** (skip what a plan already covers): break the request into
   capabilities; for each, prefer a module already installed, then core, then
   a catalog module worth installing (`list_modules({query})` →
   `get_module` → `summary`), then custom code. For every module in play,
   `get_module({module, release})` with the installed version when it is
   documented, then `get_doc` for `ai-integration` and the targeted docs the
   work needs (`extension-points`, `services`, `hooks`, `events`, `plugins`,
   `entities`, `configuration`, `permissions`, `routes`, `submodules/<x>`);
   framework APIs via `get_core_library_doc`. Keep the code / YAML / command
   examples the docs contain — they are what you build from.
3. **Outline before coding** — in your message, not in a file: modules to use
   or install (versions and docs release), the integration shape — per
   extension point, the event or hook chosen and the doc that proved it —
   and the numbered steps. A new contrib dependency is stated with its reason
   and then installed; ask only when two options differ materially and the
   choice is the user's.
4. **Install** what the outline or plan calls for: `composer require
   drupal/<name>` through the wrapper, then `drush en <name> -y`, then
   `drush cr`. If composer fails, stop and report — do not improvise a
   different module.
5. **Build step by step** from the docs' examples adapted to the project's
   names. Before writing each piece, ask what the example does *not* settle —
   the service interface and constructor arguments, the hook or event
   signature, the config schema keys, the plugin attribute fields, a route's
   parameters — and fetch **that** from the MCP with the targeted doc id and
   the same `release`. Never re-fetch a fact you already have.
6. **Follow `references/drupal-rules.md`** in every file you write — it
   overrides habit and pretraining. Custom code lives in
   `web/modules/custom/<module>/` with the layout the rules describe.
7. **Verify**: `drush cr` after every step that changes services, plugins,
   routes or config; `drush php:eval` smoke checks for what you wired; for
   config, the import → export → adopt cycle from the rules, plus a check on
   the site that fields show up in their form and view displays; the
   project's tests when present; a plan's `## Verification` commands when
   there is one. A failing check is not done: fix it, or report it plainly.
   If the docs said X and the site does Y, that is case (iii): verify against
   the source surgically, fix, and record the contradiction as a doc gap.
8. **Close**: with a plan, update it (`## Docs consulted`, `## Source reads`,
   `## Doc gaps`, deviations under `## Approach`, a `Done:` line per step).
   Always end your message with the grounding report:

   ```markdown
   ## Grounding report
   - MCP: get_module(<module>, <release>) · get_doc(<module>, <doc>) · …
   - Modules installed: none | <name> (<version>) — <why>
   - Pretraining: none | <fact> — validated via <command>
   - Source reads: none | <module> — case (i/ii/iii) — <path:line> — <why>
   - Doc gaps: none | <doc id @ release> — <what's missing/wrong>
   ```
