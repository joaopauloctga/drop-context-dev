---
name: dc-setup
description: >-
  Bootstrap Drop Context in the current Drupal project: check the drop-context
  CLI and MCP server are installed and reachable, map installed modules against
  the dropcontext.dev catalog, install the dc-* skills for the modules this
  project uses, and report what has no coverage. Use when asked to set up,
  check, or refresh Drop Context in a project, or when drop-context
  commands/tools fail with setup errors.
argument-hint: "[--no-install to only report]"
---

# /dc-setup — bootstrap Drop Context in this project

Run the checks in order; each has a clear failure message. Don't fix silently —
report what you found and what you did.

## 1. CLI present

```bash
command -v drop-context || echo "MISSING"
drop-context --version --no-update-check 2>/dev/null
```

Missing → tell the user: `composer global require joaopauloctga/drop-context`
(and ensure composer's global bin dir is on PATH). Stop here if absent.

## 2. First-run state

`drop-context skill:list --from=local --json` — if it fails with an app-setup /
"not installed" error, the CLI's one-time interactive wizard never ran. Do
**not** try to run it yourself (it needs a TTY): ask the user to run
`drop-context skill:list` once in their terminal (newer CLI versions accept
`--code-agents=.claude,.agents` to skip the prompt).

## 3. MCP reachable

Check the project's `.mcp.json` (or `claude mcp list`) for a `drop-context`
entry; then confirm the tools respond — e.g. call `list_modules` with
`query: "views"`. Not configured → offer to add to the project `.mcp.json`:

```json
{ "mcpServers": { "drop-context": { "type": "stdio", "command": "drop-context-mcp" } } }
```

## 4. Project map

```bash
drop-context project:status --json
```

If the installed CLI lacks `project:status`, build the map manually: parse
`composer.lock` for `drupal/core*` + `"type": "drupal-module"` packages, then
`get_module` per relevant module (batch only what the project actually uses).

## 5. Install skills for installed modules

For every installed module whose catalog skill (`dc-<machine, _ → ->`) exists
and is not yet installed locally:

```bash
drop-context skill:add <dc-name> --json --force
```

With `--no-install` (or if the user only asked for a check), list instead of
installing. Never remove existing skills here.

## 6. Report

End with a compact table: module | installed version | documented releases |
match (exact/line/none) | skill (installed/available/—). Then two lists:
**no coverage** (candidates for documenting with the drop-context producer
plugin, which reads the repo's own source) and **version mismatches** (docs
exist but for another line). Suggest `/dc-plan` as the next step.
