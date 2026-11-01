---
name: dc-implement
description: >-
  Implement a Drupal feature with the drop-context MCP as the source of truth:
  reads the project (composer.json / composer.lock, custom modules, config),
  matches the request against modules already installed, core, and catalog
  modules worth installing, reads their versioned docs, then installs what is
  needed, builds, verifies with drush and the project's tests, and ends with a
  grounding report. Works straight from the request; a plan file, when there
  is one, is a head start, not a prerequisite. Use when asked to implement,
  build, add, or change something in a Drupal project.
argument-hint: <what needs to be done — or the path/slug of a plan>
---

# /dc-implement: build it

Input: $ARGUMENTS

If the line above still shows a literal `$ARGUMENTS` placeholder, your client
did not substitute it: the input is the user's message that invoked this
skill.

You work in the main context so the user sees every change. Everything this
skill needs is in this file and in `references/drupal-rules.md`, next to
it. It relies on no other skill or agent. A plan, when one exists, is only
an input.

## Where facts come from

Three sources, always in this order:

1. **The project.** `composer.json` / `composer.lock` (core version, installed
   modules and their exact versions), the custom modules, the config sync
   directory, the theme, and the running site through drush. Reading and
   editing the project's own code is the job, and it is never restricted.
2. **The drop-context MCP.** Use it for every contrib module, core module, and
   core framework library the task touches. All API facts come from it:
   service IDs, hook names, events, permissions, config keys, routes, plugin
   IDs, and the code patterns that go with them.
3. **Contrib/core source, as a last resort.** Read it only in the three cases
   listed below.

Pretraining is acceptable only for core APIs absent from the catalog (Form
API, Render API, Cache API, …), and only after `drush php:eval` confirmed the
class/service/method exists on **this** site. Never use it for contrib.

## The drop-context MCP

The server is named `drop-context`, and each client prefixes its six tools
its own way: `mcp__drop-context__list_modules` (or
`mcp__plugin_drop-context-dev_drop-context__list_modules` from the plugin) in
Claude Code, `drop-context_list_modules` in OpenCode. Match them by the bare
tool names below. In Claude Code, if they are listed but not loaded, load them
with ToolSearch (query `drop-context`). If no drop-context tool exists at all,
stop and tell the user:
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
then for the targeted docs the work needs. Never call `get_doc` before
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
  the work actually needs. Never read all of them blindly.
- Never call `list_modules` without a `query`.
- Never fetch the same fact twice.
- Docs are Markdown full of code, YAML and commands. Build from those
  examples, adapted to the project's names.

## Version discipline

Align the docs release with the installed version before trusting anything.
Compare them using `available_releases` from `get_module`:

| installed vs documented | conduct |
|---|---|
| same tag | use freely |
| same release line (5.0.2 vs 5.0.3) | use; note the delta; check `release_notes` |
| different line (2.x vs 3.x) | orientation only. Verify every load-bearing fact on the site (`drush php:eval`) or by a case-(ii) source read before relying on it |
| not documented | pretraining validated on the site for core; case (i) below otherwise |

An unrecognized `release` argument makes the MCP return an **error**. It never
falls back silently, so pick a tag from `available_releases` or omit the
argument. For a module you install, use the catalog's default release and pin
`composer require` to that line.

## Reading contrib/core source: three cases only

- **(i) Not in the catalog.** `list_modules` / `get_module` confirmed the
  module is not documented.
- **(ii) Fact not in the docs.** You read the right docs from the index and the
  specific fact is genuinely absent.
- **(iii) The doc failed in practice.** You followed the doc exactly and it did
  not work: a runtime error, a failing drush command, or a failing test. The
  source is your verification. Record the contradiction as a doc gap. This is
  the most valuable feedback the catalog can get.

Say which case applies before the first read. Then read **surgically**: grep
for one symbol, and `Read` with offset/limit around the hit. Never open the
module and browse. "Just confirming", "it's faster" and "I know this module"
are not reasons. Never edit `<docroot>/modules/contrib/**`, `<docroot>/core/**`
or `vendor/**`. Extend through a custom module, using the documented extension
points.

## Input

Resolve the input before anything else:

1. **Empty.** Ask what should be built, and stop.
2. **A path to an existing file.** It is a plan.
3. **A slug.** If `.drop-context/plans/<input>.md` exists (with or without the
   `.md`), it is a plan.
4. **Anything else.** It is the request. If
   `.drop-context/plans/<kebab-slug-of-the-request>.md` exists and its
   `## Request` describes the same work, use it as the plan and say so.
   Otherwise, work from the request alone.

### Working from a plan

A plan is a fact base to start from, not ground truth. Any Markdown plan
works. Plans in the drop-context format carry these sections: `## Modules`,
`## Steps` (each step with `Files:` / `Check:`), `## Deployment`,
`## Verification`, `## Open questions`, `## Docs consulted`, and the
`<!-- module@release · doc-id -->` source comments. Use them directly.

- **Versions.** Compare the plan's module table against `composer.lock` as it
  is now.
  - A fact whose source comment names the installed release, or the same
    release line, is reused without re-fetching.
  - A fact without a source, or from a release that no longer matches, is
    re-fetched or verified on the site before you build on it.
- **Open questions.** Settle each one before the step that depends on it.
  Take the plan's assumed default when it is safe, and say so in the outline.
  Ask the user when it is not.
- **Rules.** The plan's examples still go through the rules file. When the
  two disagree, the rules win, and the deviation is recorded in the plan.

## Procedure

1. **Preflight.**
   - Make sure the MCP tools are available (see above).
   - Make sure this is a Drupal project: `composer.lock` lists `drupal/core`.
     If it does not, stop and tell the user.
   - Before the first command that writes (`en`, `cim`, `cex`, `updb`,
     `config:set`, a mutating `php:eval`), confirm with `drush status` that
     the target is the local site.
2. **Load the rules.** Read `references/drupal-rules.md` in this skill's
   directory (`${CLAUDE_SKILL_DIR}/references/drupal-rules.md` where the
   client expands that variable). If the directory is unknown, find the file
   by globbing for `dc-implement/references/drupal-rules.md`. The rules
   govern every line you write, and they override habit and pretraining.
   They cover the following:
   - events before hooks, with every hook verified through the MCP and as
     specific as possible
   - `#[Hook]` classes on core ≥ 11.1, with no business logic in hooks
   - services in `src/Services/`, registered by class name and always
     injected; another module's services injected as optional (`@?`)
   - schema for every config, and the import → export → adopt cycle
   - complete field/display config
   - drush only against the local site, and no tests unless asked
   - access and escaping, and cache metadata studied per case
   - no queues or batches unless asked
   - the comment limits of rule 10, with no comment narrating the change
   - rule 11 on drush scripts and update hooks
   - English for everything you write (rule 12)
3. **Read the project.** Run this from the directory that holds
   `composer.json`. With a plan, use it to confirm that `composer.lock` still
   matches the plan's `## Modules`.

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

   - **Wrapper.** `.ddev/` means `ddev drush` / `ddev composer` /
     `ddev exec`; `.lando.yml` means `lando drush` / `lando composer`.
     Otherwise use `vendor/bin/drush` and `composer`. Never call a global
     `drush`.
   - **Enabled modules.** When it matters, check what is **enabled**, not
     just installed:
     `drush pm:list --status=enabled --type=module --format=json`.
   - **Existing config and code.** Grep the config sync directory (usually
     `config/sync`; see `$settings['config_sync_directory']`) and the custom
     modules for the entity types, fields, routes or services the request
     touches.
4. **Discover.** Skip whatever a valid plan already settled.
   - Break the request into capabilities: the concrete things the site must
     do.
   - Match each capability, in this order of preference:
     1. a contrib module already installed (confirm with `summary` /
        `ai-integration`)
     2. core
     3. a catalog module worth installing (`list_modules({query})` with two
        or three keywords, then `get_module`, then `summary`)
     4. custom code built on the documented extension points
   - State the trade-off when two options compete.
   - For every module in play, call `get_module({module, release})`, with the
     installed version as `release` when it is documented.
   - Read `ai-integration`, then the targeted docs the work needs. For every
     module you extend, read `events` before `hooks`.
   - Read framework APIs with `get_core_library_doc`.
5. **Outline before coding.** Write it in your message, not in a file:
   - the modules to use or install, with their versions and docs release
   - the integration shape, and for each extension point the event or hook
     chosen and the doc that proved it
   - the numbered steps
   - what will ship on deploy (composer changes, config YAML, update hooks)

   Then proceed. A new contrib dependency is stated with its reason, then
   installed. Stop to ask only for decisions that belong to the user:
   - two options that differ materially
   - a rule-11 choice that is unclear
   - a plan open question without a safe default
6. **Install** what the outline calls for, through the wrapper:
   `composer require drupal/<name>:<constraint>`, then `drush en <name> -y`,
   then `drush cr`. If composer fails, stop and report. Do not improvise a
   different module.
7. **Build step by step**, from the docs' examples adapted to the project's
   names. Custom code lives in `<docroot>/modules/custom/<module>/`, with the
   layout the rules describe. Before writing each piece, find what the example
   does *not* settle: the service interface and constructor arguments, the
   hook or event signature, the config schema keys, the plugin attribute
   fields, or a route's parameters. Fetch **that** from the MCP with the
   targeted doc id and the same `release`. After each step:
   - Run `drush cr` when it changed services, plugins, hooks, routes or
     config.
   - Run the step's check: the plan's `Check:` line, or a `drush php:eval`
     smoke test of what you wired.
   - Fix a failing check before moving on.
8. **Verify end to end.**
   - For config, run the import → export → adopt cycle from the rules.
   - On the site, check that fields show up in their form and view displays.
   - Run the project's tests and phpcs when they exist.
   - Run the plan's `## Verification` commands when there is a plan.
   - A failing check means the work is not done. Fix it, or report it
     plainly.
   - If the docs said X and the site does Y, that is case (iii). Verify
     against the source surgically, fix, and record the contradiction.
9. **Make it deployable (rules 2 and 11).**
   - Every piece of config the site needs is exported and lives in the config
     sync directory or the module's `config/install` / `config/optional`.
   - What a local drush script did ships as YAML or as an update hook. When
     it is unclear which, or where it lives, ask the user.
   - Every module the code uses is listed under `dependencies:` in
     `.info.yml`.
   - `composer.json` / `composer.lock` carry the new requirements.
10. **Close.**
    - When working from a plan, update it:
      - set `Status:` to `implemented — <YYYY-MM-DD>` (or
        `partial — steps 1–n`)
      - add a `Done:` line under each finished step
      - append to `## Docs consulted`, `## Source reads` and `## Doc gaps`
      - record each deviation under `## Approach`
    - Always end your message with the grounding report:

    ```markdown
    ## Grounding report
    - MCP: get_module(<module>, <release>) · get_doc(<module>, <doc>) · …
    - Modules installed: none | <name> (<version>) — <why>
    - Pretraining: none | <fact> — validated via <command>
    - Source reads: none | <module> — case (i/ii/iii) — <path:line> — <why>
    - Doc gaps: none | <doc id @ release> — <what's missing/wrong>
    - Deploy: <what ships — config, update hooks, composer changes>
    ```

## Anti-fabrication

Never invent a service ID, hook name, event name, route, permission string,
config key, or plugin ID. Each one comes from a cited doc or is verified on
the site (`drush php:eval`, config inspection) before you rely on it. When
nothing confirms a fact, give an explicit "not verified" note or ask the user.
Never fill the gap with a plausible guess. Leaving a fact out is better than
inventing it.
