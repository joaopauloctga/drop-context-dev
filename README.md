# drop-context-dev — a Drupal developer agent grounded in the drop-context MCP

A Claude Code plugin that ships the **Drop Context Developer**: a Drupal
developer agent that plans and implements features by reading your project
(`composer.json`, `composer.lock`, custom modules, config) and the versioned
module documentation served by the **drop-context MCP** — for the releases you
actually run — instead of reading contrib/core source or trusting pretraining.
Source reading is a documented last resort, never the default.

Why: fewer tokens per feature, and facts that match the installed release.

One agent, two skills. The skills are independent: each carries its own copy
of the grounding discipline (sources of facts, MCP tools, version discipline,
source-read cases, anti-fabrication) and of the Drupal rules, so either one
works without the other.

| skill | what it does |
| --- | --- |
| `/dc-plan <what needs to be done>` | Maps the project, matches the request against modules already installed, core, and modules in the catalog worth installing, reads their docs, and writes a self-contained plan to `.drop-context/plans/<slug>.md`. The plan holds concrete code and config examples, a `Check:` per step, and deploy notes, so a person or any agent can build from it. Runs in a forked context — only the path and a short summary come back. |
| `/dc-implement <what needs to be done>` | Implements the request in the main context with the MCP as the source of truth: reads the project, matches modules (installed, core, or worth installing), reads their docs, installs what is needed, builds, verifies with drush and tests, makes it deployable, and ends with a grounding report. Any plan file (from `/dc-plan` or written by hand) is an optional head start: its facts are re-checked against the installed versions, never trusted blindly. |

The `dc-*` skills published on dropcontext.dev (installed with the
`drop-context` CLI) are for **you**, the human developer. The agent does not
use or manage them — everything it knows about a module comes from the MCP.

## Installation

```text
/plugin marketplace add joaopauloctga/drop-context-dev
/plugin install drop-context-dev@drop-context-dev
```

To update later:

```text
/plugin marketplace update drop-context-dev
/plugin update drop-context-dev
```

### Cursor, Codex, OpenCode (and Claude Code without the plugin)

The drop-context CLI installs this repository into a Drupal project for every
coding agent it uses — skills, the agent in each client's own format, and the
MCP server registration:

```bash
cd <drupal-project>
drop-context dev:install                          # coding agents found in the project
drop-context dev:install --code-agents=.cursor    # or pick them
drop-context dev:remove
```

See the [CLI README](https://github.com/joaopauloctga/drop-context#install-the-drop-context-developer)
for what goes where. Invoke the skills as `/dc-plan` and `/dc-implement`
(`$dc-plan` / `$dc-implement` in Codex).

## Prerequisites

The plugin bundles an `.mcp.json` that registers the `drop-context-mcp` stdio
server, but the server itself ships with the **drop-context CLI**:

```bash
composer global require joaopauloctga/drop-context
which drop-context-mcp        # composer's global bin dir must be on PATH
```

To register the server yourself instead of relying on the bundled `.mcp.json`:

```bash
claude mcp add drop-context --scope project -- drop-context-mcp
```

Nothing else to set up: no wizard, no per-project state, no skills to install
for the agent.

## Typical flow

```text
# straight to code
/dc-implement add a bookmark flag on articles and show the count in teasers
   → code + verification + grounding report

# or plan first, review, then build from the plan
/dc-plan add a bookmark flag on articles and show the count in teasers
   → .drop-context/plans/bookmark-flag-on-articles.md
/dc-implement bookmark-flag-on-articles
   → code + verification + grounding report
```

Every task ends with a **grounding report** — which MCP docs grounded the
work, which modules were proposed or installed, any pretraining facts and how
they were validated, any source reads (with their justification case), any
doc gaps found, and what ships on deploy. A plan carries the same information
in its last sections. Doc gaps feed the producer side: the sibling
[drop-context plugin](https://github.com/joaopauloctga/drop-context-plugin) can
document an uncovered module straight from your repo's source.

## What's inside

| path | what it is |
| --- | --- |
| `agents/drop-context-developer.md` | The agent `/dc-plan` forks into, also usable directly: where facts come from, how to read the project, the MCP tools and doc ids, module matching, version discipline, a baseline of Drupal conduct, the grounding report. A skill's own procedure and rules take precedence over it. |
| `skills/dc-plan/` | `/dc-plan` — forked planning run. Self-contained: owns the plan format and the procedure. |
| `skills/dc-implement/` | `/dc-implement` — implements a request in the main context, directly or from any plan. Self-contained. |
| `skills/*/references/drupal-rules.md` | The Drupal rules, one identical copy per skill: events before hooks, verified and specific hooks (`#[Hook]` on core ≥ 11.1), no logic in hooks, services by class name and injected (optional `@?` for other modules' services, deploy-safe), config schema + import/export cycle, access, cacheability per case, no queues/batches or tests unless asked, strict comment lengths with usage-documenting class docblocks, drush scripts turned into deployable artifacts, English only. |
| `.mcp.json` | Registers `drop-context-mcp` (stdio) for projects using the plugin. |

Content is duplicated between the skills on purpose. When you change a shared
part, change it in both places. The rules files must stay byte-identical:

```bash
cmp skills/dc-plan/references/drupal-rules.md skills/dc-implement/references/drupal-rules.md
```

The shared sections of the two `SKILL.md` files ("Where facts come from",
"The drop-context MCP", "Version discipline", "Reading contrib/core source",
"Anti-fabrication") are written to read the same, apart from small
differences of context. Diff them when you change one.

## Releases and versioning

The plugin carries a plain semver `version` in `.claude-plugin/plugin.json`,
and each release is marked by a git tag of the form `{name}--v{version}` —
`drop-context-dev--v0.1.0`. The tag is what a marketplace resolves an installed
version against, so the two must not drift. `claude plugin tag` creates it, and
refuses to if `plugin.json` and the enclosing marketplace entry disagree:

```bash
claude plugin validate .            # manifests well-formed?
claude plugin tag --dry-run         # what would be tagged, without tagging
claude plugin tag --push            # tag HEAD and push it to origin
```

`tag` also refuses on a dirty working tree, so a tag always points at committed
state. Cutting a release is therefore: bump `version` in `plugin.json`, commit,
`claude plugin tag --push`.

## Relation to the drop-context producer plugin

Two plugins, two directions:

- **drop-context** (producer): reads Drupal source in your repo → writes docs
  and `dc-*` skills for humans. Zero network.
- **drop-context-dev** (this, consumer): reads the published catalog through
  the MCP → plans and builds features. Networked by nature; depends on the CLI
  only for the `drop-context-mcp` binary.

## Local development

This directory is the plugin — `.claude-plugin/marketplace.json` sits right
here, so a local checkout can be added as a marketplace and installed from
disk, with no publishing step:

```text
/plugin marketplace add /absolute/path/to/developer
/plugin install drop-context-dev@drop-context-dev
```

Re-run `/plugin marketplace update drop-context-dev` after editing the manifests.

To test in a Drupal project without going through the plugin system at all,
symlink instead:

```bash
ln -s <here>/agents/drop-context-developer.md <project>/.claude/agents/
for s in dc-plan dc-implement; do
  ln -s <here>/skills/$s <project>/.claude/skills/$s
done
```
