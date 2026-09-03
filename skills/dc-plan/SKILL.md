---
name: dc-plan
description: >-
  Plan a Drupal feature with the drop-context MCP: reads composer.json /
  composer.lock to map the installed core and modules, matches what the request
  needs against modules already installed and modules in the catalog worth
  installing, reads their versioned docs, and writes a plan full of concrete
  code and config examples to .drop-context/plans/<slug>.md. Use when asked to
  plan, scope, or design a Drupal feature or change before implementing it.
  Runs the drop-context-developer agent in a forked context — only the plan
  path and a short summary come back.
context: fork
agent: drop-context-developer
argument-hint: <what needs to be done>
---

# /dc-plan — write the plan

You are planning, not implementing: **do not create or edit project code or
config in this run.** Your only write is the plan file (and its directory).

The request is the invocation argument. If it is empty or ambiguous, choose the
most useful interpretation, state it under `## Request`, and continue — do not
stop to ask.

## Rules for the examples

Before writing any example, read `../dc-implement/references/drupal-rules.md`
— the rules file shared with /dc-implement, one directory up from this
skill's `SKILL.md` (if the relative path does not resolve, locate it with Glob:
`**/dc-implement/references/drupal-rules.md`). Every example in the plan must
already follow it: events before hooks, hooks verified through the MCP and as
specific as possible, no business logic in hooks, services in `src/Services/`
registered by class name and injected, services of other modules injected as
optional (`@?`) so a deploy never breaks on a module not yet enabled, schema
for every config, complete field/display config, access on every route, cache
metadata only where the case demands it, no queues or batches unless asked,
no tests unless asked, comments in the examples within the limits of rule 10
(short inline comments, short method docblocks, class docblocks that show
usage with `@code`), and — rule 11 — every step that runs a drush script to
create config or data names the deployable artifact that replaces it
(exported YAML, `hook_post_update_NAME`, `hook_deploy_NAME`), or an open
question for the user when that is not clear. The plan itself and every
example in it are written in English, whatever language the request came in
(rule 12).

## Procedure

1. **Read the project** (agent section "Reading the project"): core version,
   installed modules with versions, custom modules, command wrapper, and the
   existing config/code the request obviously touches.
2. **Break the request into capabilities** — the concrete things the site must
   be able to do (store X, react to Y, expose Z, display W). Three to eight
   lines; this is the checklist the rest of the plan answers.
3. **Match modules to capabilities** (agent section "Matching modules to the
   request"): installed first, then core, then `list_modules({query})` for
   catalog modules worth installing, then custom code for the rest. Call
   `get_module({module, release})` for every module that stays in the plan —
   `release` = the installed version when it is documented ("Version
   discipline"); otherwise omit it and record the mismatch.
4. **Read the docs**: per module `get_doc({module, release})` — that is
   `ai-integration` — then the targeted docs the steps need, picked from the
   `docs` index by id, description and size (`extension-points`, `services`,
   `hooks`, `events`, `plugins`, `entities`, `configuration`, `permissions`,
   `routes`, `submodules/<x>`); core framework APIs via
   `get_core_library_doc({library})`. For every module you will extend, read
   `events` before `hooks` and record which one you chose and why. Collect
   every fact you will rely on and keep the code / YAML / command examples
   the docs contain — you will paste them into the steps.
5. **Decide the approach** — integration shape (config-only, plugin, event
   subscriber, hook, service decoration; new custom module vs. an existing
   one), config schema, permissions, what gets installed. Prefer the shapes
   the docs recommend in `ai-integration` / `extension-points`; spell out the
   trade-offs.
6. **Write the plan** to `<project>/.drop-context/plans/<slug>.md` with the
   exact section list from the agent definition ("The plan"). Requirements:
   - `## Modules`: one row per module — installed version, docs release used,
     match (exact / line / different / undocumented), role (`use` / `core` /
     `install`), and why. For `install` rows the exact `composer require` and
     `drush en` commands go in the first step.
   - `## Approach`: per extension point, whether an event or a hook was
     chosen and the doc that proved it exists; which logic goes into which
     service.
   - `## Steps`: numbered and ordered so each step leaves the site working.
     Each step names its files and carries the **concrete code and config** it
     needs — event subscribers or `src/Hook/` classes with `#[Hook(...)]`;
     services in `src/Services/` with constructor injection and their
     `.services.yml` entry by class name; `.info.yml`; `config/install` YAML
     with `dependencies:` and its `config/schema` entry; field storage, field
     and display config; `*.routing.yml` with access; `*.permissions.yml`;
     drush commands — adapted from the docs to the project's names, each
     example followed by its source comment
     (`<!-- module@release · doc-id -->`). Generous examples are the goal:
     /dc-implement should be able to build from the plan with few extra doc
     reads.
   - `## Verification`: concrete commands with the detected wrapper
     (`ddev drush cr`, `ddev drush php:eval '…'`, `ddev drush config:get …`,
     phpunit when present).
   - `## Docs consulted`: one line per MCP call, in order.
   - `## Source reads` / `## Doc gaps`: `none` unless genuinely needed.
7. **Return** — final message only: the plan path, then at most 10 lines
   (interpretation, modules to use / install, chosen shape, step count, open
   questions). No file dumps.
