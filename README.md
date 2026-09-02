# drop-context-developer — a grounded Drupal developer agent

A Claude Code plugin that ships the **Drop Context Developer**: a Drupal
developer agent that plans and implements features by consulting the
[dropcontext.dev](https://dropcontext.dev) catalog — installed `dc-*` skills
and the `drop-context-mcp` documentation tools — for the **release you
actually run**, instead of reading contrib/core source or trusting
pretraining. Source reading is a documented last resort (module not in the
catalog; fact missing from the docs; the doc was followed and didn't work),
never the default.

Why: fewer tokens per feature, and facts that match the installed release.

## Prerequisites

- The **drop-context CLI** (provides the `drop-context-mcp` stdio server and
  `skill:add`):

  ```bash
  composer global require joaopauloctga/drop-context
  drop-context skill:list        # first run is interactive (picks agent dirs)
  ```

- The MCP server registered for the project (the plugin bundles this
  `.mcp.json`; or add it yourself):

  ```bash
  claude mcp add drop-context --scope project -- drop-context-mcp
  ```

## What's inside

| path | what it is |
| --- | --- |
| `agents/drop-context-developer.md` | The agent: the source ladder (skill → MCP → validated pretraining → source-as-last-resort), version discipline, token budgets, brief contract, grounding report. |
| `skills/dc-plan/` | `/dc-plan <feature>` — runs the agent in a **forked context**; consults docs there and writes a Context Brief to `.drop-context/briefs/<slug>.md`. Only the path + a short summary return. |
| `skills/dc-implement/` | `/dc-implement <brief>` — executes the brief in the main context, verifying with drush/phpunit, ending with a grounding report. |
| `skills/dc-setup/` | `/dc-setup` — checks CLI + MCP, maps installed modules against the catalog, installs the matching `dc-*` skills. |
| `skills/dc-consult/` | The condensed source ladder for the main context / small tasks. |
| `.mcp.json` | Registers `drop-context-mcp` (stdio) for projects using the plugin. |

## Typical flow

```text
/dc-setup                                   # once per project
/dc-plan   add a bookmark flag on articles  # → .drop-context/briefs/bookmark-flag-on-articles.md
/dc-implement bookmark-flag-on-articles     # → code + verification + grounding report
```

Every task ends with a **grounding report** — which skills/MCP docs grounded
the work, any pretraining facts and how they were validated, any source reads
(with their justification case), and any doc gaps found. Doc gaps feed the
producer side: the sibling [drop-context plugin](https://github.com/joaopauloctga/drupal-context-ai)
can document an uncovered module straight from your repo's source.

## Relation to the drop-context producer plugin

Two plugins, two directions:

- **drop-context** (producer): reads Drupal source in your repo → writes docs
  and `dc-*` skills. Zero network.
- **drop-context-developer** (this, consumer): reads the published catalog via
  MCP/CLI → builds features. Networked by nature; depends on the CLI.

## Local development

This directory is the plugin. To test in a Drupal project without installing
the plugin, symlink:

```bash
ln -s <here>/agents/drop-context-developer.md <project>/.claude/agents/
for s in dc-consult dc-plan dc-implement dc-setup; do
  ln -s <here>/skills/$s <project>/.claude/skills/$s
done
```
