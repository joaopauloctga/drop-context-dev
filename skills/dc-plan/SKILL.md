---
name: dc-plan
description: >-
  Plan a Drupal feature with the drop-context MCP as the source of truth:
  reads composer.json / composer.lock to map the installed core and modules,
  matches what the request needs against modules already installed, core, and
  catalog modules worth installing, reads their versioned docs, and writes a
  self-contained plan — concrete code and config examples, a check per step,
  deploy notes — to .drop-context/plans/<slug>.md. Use when asked to plan,
  scope, or design a Drupal feature or change before building it. Runs in a
  forked context; only the plan path and a short summary come back.
context: fork
agent: drop-context-developer
argument-hint: <what needs to be done>
---

# /dc-plan: write a plan anyone can build from

Request: $ARGUMENTS

If the line above still shows a literal `$ARGUMENTS` placeholder, your client
did not substitute it: the request is the user's message that invoked this
skill. In Claude Code this skill runs forked into the `drop-context-developer`
agent; other clients may run it inline. The procedure is the same either way.

You are planning, not implementing. The only file you write is the plan (and
its directory). Do not create or edit project code or config. Do not run
`composer require`, `drush en`, `cim`, `cex`, `cr`, or a `php:eval` that
writes. Read-only drush is fine: `status`, `pm:list`, `config:get`, and a
`php:eval` that only reads.

The plan is the whole hand-off. Whoever builds it (a person, /dc-implement,
another agent) starts with only this file and the project. Put everything they
need in the plan: versions, the docs release behind each fact, the code and
config, the commands, and each decision with its reason. Everything this run
needs is in this file and in `references/drupal-rules.md`, next to it. It
relies on no other skill.

- **Empty request:** return one line asking what to plan, and do nothing else.
- **Ambiguous request:** pick the most useful interpretation, write it under
  `## Request`, and continue. You cannot ask the user anything from here. Put
  each decision that belongs to the user under `## Open questions`, with the
  default the plan assumed so the build can proceed.

## Where facts come from

Three sources, always in this order:

1. **The project.** `composer.json` / `composer.lock` (core version, installed
   modules and their exact versions), the custom modules, the config sync
   directory, the theme, and the running site through read-only drush.
2. **The drop-context MCP.** Use it for every contrib module, core module, and
   core framework library the request touches. All API facts come from it:
   service IDs, hook names, events, permissions, config keys, routes, plugin
   IDs, and the code patterns that go with them.
3. **Contrib/core source, as a last resort.** Read it only in the three cases
   listed below.

Pretraining is acceptable only for core APIs absent from the catalog (Form
API, Render API, Cache API, …), and only after a read-only
`drush php:eval` confirmed the class/service/method exists on **this** site.
Never use it for contrib.

## The drop-context MCP

The server is named `drop-context`, and each client prefixes its six tools
its own way: `mcp__drop-context__list_modules` (or
`mcp__plugin_drop-context-dev_drop-context__list_modules` from the plugin) in
Claude Code, `drop-context_list_modules` in OpenCode. Match them by the bare
tool names below. In Claude Code, if they are listed but not loaded, load them
with ToolSearch (query `drop-context`). If no drop-context tool exists at all,
stop and return:
`drop-context MCP unavailable — install the CLI (composer global require
joaopauloctga/drop-context) and make sure drop-context-mcp is on PATH.`

The parameter names below are exact. `get_module` and `get_doc` take
**`module`**, which is the `machine_name` value from `list_modules`. Do not
pass a parameter named `machine_name`.

| tool | arguments | returns |
|---|---|---|
| `list_modules` | `query` (substring — always pass one), `core_only?`, `limit?`, `offset?` | rows with `machine_name`, `title`, `is_core`, `summary`, `default_release`, `releases` |
| `get_module` | `module`, `release?` | metadata, `available_releases` (each with `release_line`), `release_notes`, the `docs` index (`id`, `title`, `category`, `description`, `size`) |
| `get_doc` | `module`, `doc?` (default `ai-integration`), `release?` | one doc, Markdown |
| `list_core_libraries` | `query?`, `kind?` (`core` \| `component`) | rows with `id`, `qualified_name`, `kind`, `summary`, `core_version` |
| `get_core_library` | `library` (`Core/Queue`, `drupal.core.queue`, or the PHP namespace) | metadata + docs index |
| `get_core_library_doc` | `library`, `doc?` (default `usage`) | one doc, Markdown |

For each module, call `get_module` first. Pick docs from its `docs` index by
`id`, `description` and `size`. Then call `get_doc` for `ai-integration`,
then for the targeted docs the steps need. Never call `get_doc` before
`get_module`.

| doc id | use it for |
|---|---|
| `ai-integration` | the implementation guide written for agents — start here, always |
| `summary` | what the module is — enough to decide whether it fits |
| `extension-points` | the sanctioned ways to extend it: plugins, events, hooks, decoration |
| `services` | service IDs, interfaces, constructor signatures |
| `hooks` | hooks the module invokes and implements, with signatures |
| `events` | event names, event classes, subscriber examples |
| `plugins` | plugin types, attributes/annotations, base classes |
| `entities` | entity types, bundles, fields, storage |
| `configuration` | config objects, schema, keys and defaults |
| `permissions` / `routes` | exact permission strings and route names |
| `submodules/<name>` | one submodule, condensed |

Core libraries (`Core/Ajax`, `Core/Batch`, `Core/Queue`, `Core/Hook`, …):
`usage` is the verified-patterns guide, `api` is the reference,
`architecture` covers the internals, and `topics/<slug>` holds deeper dives.

Budget:

- One doc is about 2–4k tokens, and a whole module is about 20k.
- For each module, read `ai-integration` plus the two to four targeted docs
  the steps actually need. Never read all of them blindly.
- Never call `list_modules` without a `query`.
- Never fetch the same fact twice.
- Docs are Markdown full of code, YAML and commands. Keep those examples,
  because they become the plan's steps.

## Version discipline

Align the docs release with the installed version before trusting anything.
Compare them using `available_releases` from `get_module`:

| installed vs documented | conduct |
|---|---|
| same tag | use freely |
| same release line (5.0.2 vs 5.0.3) | use; note the delta; check `release_notes` |
| different line (2.x vs 3.x) | orientation only. Mark every fact "unverified on installed" and list the load-bearing ones under `## Open questions` for the build to verify |
| not documented | pretraining validated on the site for core; case (i) below otherwise |

An unrecognized `release` argument makes the MCP return an **error**. It never
falls back silently, so pick a tag from `available_releases` or omit the
argument. For a module the plan installs, use the catalog's default release
and pin `composer require` to that line.

## Reading contrib/core source: three cases only

- **(i) Not in the catalog.** `list_modules` / `get_module` confirmed the
  module is not documented.
- **(ii) Fact not in the docs.** You read the right docs from the index and the
  specific fact is genuinely absent.
- **(iii) The doc contradicts the site.** A read-only check on the site
  disagrees with the doc. Record the contradiction under `## Doc gaps`. This
  is the most valuable feedback the catalog can get.

Record the case under `## Source reads` before the first read. Then read
**surgically**: grep for one symbol, and `Read` with offset/limit around the
hit. Never open the module and browse. "Just confirming", "it's faster" and
"I know this module" are not reasons.

## Procedure

1. **Preflight.** Make sure the MCP tools are available (see above). Make sure
   this is a Drupal project: `composer.lock` lists `drupal/core`. If it does
   not, return one line saying so.
2. **Load the rules.** Read `references/drupal-rules.md` in this skill's
   directory (`${CLAUDE_SKILL_DIR}/references/drupal-rules.md` where the
   client expands that variable). If the directory is unknown, find the file
   by globbing for `dc-plan/references/drupal-rules.md`. Every decision and
   every example in the plan follows these rules. They cover the following:
   - events before hooks, with every hook verified through the MCP and as
     specific as possible
   - `#[Hook]` classes on core ≥ 11.1, with no business logic in hooks
   - services in `src/Services/`, registered by class name and injected;
     another module's services injected as optional (`@?`) so a deploy never
     breaks
   - schema for every config, and complete field/display config
   - access on every route, and cache metadata only where correctness needs it
   - no queues, batches or tests unless asked
   - the comment limits of rule 10
   - rule 11 on drush scripts and update hooks
   - English for everything (rule 12)
3. **Read the project.** Run this from the directory that holds
   `composer.json`:

   ```bash
   python3 - <<'PY'
   import glob, json
   cj = json.load(open('composer.json'))
   web = (cj.get('extra', {}).get('drupal-scaffold', {}).get('locations', {})
          .get('web-root', 'web/')).strip('/') or '.'
   lock = json.load(open('composer.lock'))
   pk = lock.get('packages', []) + lock.get('packages-dev', [])
   print('docroot', web)
   print('core', next((p['version'] for p in pk if p['name'] == 'drupal/core'), '?'))
   for p in sorted(pk, key=lambda p: p['name']):
       if p.get('type') == 'drupal-module':
           print('module', p['name'].split('/', 1)[1], p['version'].lstrip('v'))
   for info in sorted(glob.glob(f'{web}/modules/custom/**/*.info.yml', recursive=True)):
       print('custom', info)
   PY
   test -d .ddev && echo "wrapper: ddev"; test -f .lando.yml && echo "wrapper: lando"
   ```

   - **Wrapper.** `.ddev/` means `ddev drush` / `ddev composer`; `.lando.yml`
     means `lando drush` / `lando composer`. Otherwise use
     `vendor/bin/drush` and `composer`. Never call a global `drush`.
   - **Enabled modules.** When it changes the plan, check what is
     **enabled**, not just installed:
     `drush pm:list --status=enabled --type=module --format=json`.
   - **Existing config and code.** Grep the config sync directory (usually
     `config/sync`; see `$settings['config_sync_directory']`) and the custom
     modules for the entity types, fields, routes or services the request
     touches. Reuse existing custom modules when one fits.
4. **Break the request into capabilities.** These are the concrete things the
   site must be able to do: store X, react to Y, expose Z, display W. Write
   three to eight lines. They become `## Capabilities`, the checklist the rest
   of the plan answers.
5. **Match each capability**, in this order of preference:
   1. **Already installed.** A contrib module in `composer.lock` provides it.
      Confirm with its `summary` / `ai-integration`.
   2. **Core.** A core module or framework library provides it.
   3. **In the catalog, not installed.** Call `list_modules({query})` with two
      or three keywords from the request, and read the `summary` of the
      plausible hits through `get_module`. Propose the best fit and say why.
      Prefer modules whose documented release matches the site's core.
   4. **Custom code.** Use it only for what nothing above covers, built on the
      documented extension points of the modules involved.

   When two options compete (a new dependency vs. more custom code, for
   instance), state the trade-off. For every module that stays in the plan,
   call `get_module({module, release})`. Use the installed version as
   `release` when it is documented. Otherwise follow "Version discipline".
6. **Read the docs.**
   - For each module, call `get_doc` for `ai-integration`, then for the
     targeted docs the steps need. Use the same `release` throughout.
   - Read core framework APIs with `get_core_library_doc`.
   - For every module you extend, read `events` before `hooks`, and record
     which one you chose and why.
   - Collect every fact you will rely on, and keep the docs' code, YAML and
     command examples.
7. **Decide the approach.**
   - Pick the integration shape: config-only, plugin, event subscriber, hook,
     or service decoration.
   - Pick where the code lives: a new custom module or an existing one.
   - Settle the config schema, the permissions, and what gets installed.
   - Prefer the shapes that `ai-integration` / `extension-points` recommend,
     and spell out the trade-offs.
   - Decide what ships on deploy, following rules 2 and 11: composer changes,
     config YAML, and update hooks.
8. **Write the plan** to `<project>/.drop-context/plans/<slug>.md`, using the
   format below. The slug is the kebab-case of the request in English, at
   most six words. If that file already exists, do not overwrite it: use
   `<slug>-2.md` (then `-3`, …).
9. **Self-check before returning.** Fix the plan until every answer is yes:
   - Does every capability map to a module or to custom code?
   - Does every service ID, hook, event, route, permission, config key and
     plugin ID come from a cited doc or a site check?
   - Does every step name its files, carry its code/config, and end with a
     `Check:`?
   - Does every example follow the rules file, comments included?
   - Does every drush script that creates config or data name the deployable
     artifact that replaces it, or an open question?
   - Do all commands use the detected wrapper?
   - Is everything in English?
10. **Return.** The final message holds only the plan path, then at most 10
    lines: the interpretation, the modules to use / install, the chosen
    shape, the step count, the open questions, and any doc gaps. No file
    dumps.

## Plan format

Keep the section names and their order exactly as below. Every section is
present. Write `none` when one is empty.

````markdown
# Plan: <feature>

Status: draft — <YYYY-MM-DD>

## Request
What was asked, and the interpretation chosen if it was ambiguous.

## Project
Core version · docroot · wrapper · custom modules involved · the existing
config and code the request touches (paths).

## Capabilities
1. <capability> → <module (use/core/install) | custom: <module>/<class>>

## Modules
| module | installed | docs release | match | role | why |
|---|---|---|---|---|---|
| flag | 5.0.2 | 5.0.3 | line | use | provides the bookmark flag type |

match: exact / line / different / undocumented. role: `use` (installed),
`core`, or `install` (new; its `composer require` + `drush en` go in step 1).

## Approach
The integration shape and the decisions behind it. For each extension
point: event or hook, and the doc that proved it exists. Which logic goes
into which service. The trade-offs that were weighed.

## Steps
### 1. <title>
- Files: `web/modules/custom/<module>/…`
- Serves: capability <n>
- Check: `ddev drush …` (what it must show)

```php
// the code, adapted to the project's names
```
<!-- flag@5.0.3 · events -->

## Deployment
What ships and what runs where: composer.json/lock changes, config YAML
(config/sync or the module's config/install), update hooks
(`hook_post_update_NAME` / `hook_deploy_NAME`) and what each does, and the
optional `@?` injections that keep the container compiling before
`drush deploy` enables new modules. Each local drush script, and the
artifact that replaces it.

## Verification
End-to-end commands with the detected wrapper (`ddev drush cr`,
`ddev drush php:eval '…'`, `ddev drush config:get …`, phpunit or phpcs when
the project ships them), each with its expected result.

## Open questions
- <question> (assumed: <default the plan follows>)

## Docs consulted
- get_module(flag, 5.0.3): releases, docs index
- get_doc(flag, events, 5.0.3): FlagEvents::ENTITY_FLAGGED, subscriber example

## Source reads
none | <module> — case (i/ii/iii) — <path:line> — <why>

## Doc gaps
none | <doc id @ release> — <what was missing or wrong>
````

Examples are the point of the plan:

- **Concrete.** Every step that touches a module API carries a concrete
  example: a PHP class, `.services.yml`, `.info.yml`, `config/install` YAML
  with `dependencies:` and its `config/schema` entry, field storage/field/
  display config, routing with access, permissions, or drush commands.
- **Adapted and sourced.** Each example is adapted to the project's names and
  followed by its source (`<!-- module@release · doc-id -->`). Code that no
  doc dictates says where it comes from: `<!-- rules §2 -->` or
  `<!-- project: path -->`.
- **Explained when missing.** A step without an example says why: nothing in
  the docs, to be resolved during the build.
- **Generous.** Write enough that the build needs few extra doc reads.

## Anti-fabrication

Never invent a service ID, hook name, event name, route, permission string,
config key, or plugin ID. Each one comes from a cited doc or from a read-only
check on the site. When nothing confirms a fact, write it down as "not
verified" and add it to `## Open questions`. Never fill the gap with a
plausible guess. Leaving a fact out is better than inventing it, because the
plan may be built without review.
