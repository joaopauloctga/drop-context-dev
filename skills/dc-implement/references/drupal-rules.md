# Drupal rules for generated examples and implementation

Loaded by /dc-plan (every example in a plan must already follow these) and by
/dc-implement (every line of code and config written must follow these). Where
a rule says "check the MCP", that means `get_doc` / `get_core_library_doc` on
the drop-context MCP — never memory. Custom modules use the layout below.

```text
web/modules/custom/<module>/
  <module>.info.yml                 core_version_requirement, package, dependencies
  <module>.services.yml             services registered by class name
  <module>.permissions.yml / <module>.routing.yml
  config/install/  config/optional/  config/schema/<module>.schema.yml
  src/Hook/                          #[Hook] classes — no <module>.module file
  src/Services/                      business logic, injected everywhere
  src/Event/  src/EventSubscriber/   events you dispatch / subscribe to
  src/Plugin/ src/Controller/ src/Form/ src/Entity/
```

## 1. Extending behaviour: events first, then hooks

- **Events over hooks.** Before choosing a hook, read the module's `events`
  doc (`get_doc({module, doc: "events"})`). If the module dispatches an event
  that covers the need, subscribe to it. Only when no event fits, read
  `hooks` and pick one — and say in the plan which doc decided it.
- **Never assume a hook exists.** Every hook name comes from the MCP: the
  module's `hooks` doc; for hooks that core provides (entities, fields,
  forms, theme and preprocess, cron, …) the core module that owns them
  (`node`, `field`, `system`, `views`, …) through the same `get_module` /
  `get_doc` flow; and `get_core_library_doc({library: "Core/Hook"})` for the
  attribute mechanism itself — what `#[Hook]` supports on the installed core,
  how preprocess and ordering work.
- **Be as specific as the hook system allows.** `hook_form_FORM_ID_alter`
  over `hook_form_alter`; `hook_ENTITY_TYPE_presave` (`node_presave`) over
  `hook_entity_presave`, with an early bundle guard (entity hooks specialise
  by type, not by bundle); the most specific preprocess variant the installed
  core supports (`preprocess_node__article`) over `preprocess_node`. For a
  contrib module's own hooks, the module's `hooks` doc says which specialised
  variants exist (per type, per plugin, per bundle, per field …) — use the
  most specific one it lists. The generic hook is acceptable only when the
  behaviour is genuinely identical for every variation — say so in a comment.
- **Hooks carry no business logic.** A hook adapts or extends behaviour that
  already exists. Anything beyond that — external API calls, loading entities
  unrelated to the hook's subject, computation that is not the hook's own
  purpose — lives in a service injected into the hook class; the hook only
  calls it.
- **Hook attributes only.** `#[Hook('hook_name')]`
  (`Drupal\Core\Hook\Attribute\Hook`) on a method of a class under
  `src/Hook/`, with dependencies injected through the constructor. Never
  create or extend a `<module>.module` file. Confirm discovery and
  registration details for the installed core in the `Core/Hook` docs.
- **Exposing your own extension point.** When a feature of your module is
  meant to be extended by other modules, dispatch an event — a dedicated
  event class under `src/Event/` with typed getters/setters and a name
  constant — instead of inventing a hook.

## 2. Services and dependency injection

- Services live in `src/Services/`. The service ID is the fully-qualified
  class name, so consumers refer to it as `Foo::class`:

  ```yaml
  # my_module.services.yml
  services:
    Drupal\my_module\Services\OrderPricing:
      autowire: true
    Drupal\my_module\Services\MenuSync:
      autowire: true
      arguments:
        $logger: '@logger.channel.my_module'
  ```

  Type-hint constructor arguments with interfaces
  (`EntityTypeManagerInterface`, `ConfigFactoryInterface`, …) so autowiring
  resolves them; add explicit `arguments:` only for what autowiring cannot
  resolve (logger channels, parameters, one of several implementations).
- Always inject. `\Drupal::service()`, `\Drupal::entityTypeManager()` and
  friends are forbidden inside classes; the only exception is a static
  context that cannot receive injection (a static callback, a static
  factory). Plugins get injection through
  `ContainerFactoryPluginInterface::create()`; controllers and forms through
  `ControllerBase` / `FormBase::create()`.
- One responsibility per service, named for what it does (`OrderPricing`,
  `MenuSync`) — not `Helper`, `Manager`, `Utils`.
- **Depending on another module's service must not break the deploy.** On
  deploy, drush rebuilds the container before config import enables new
  modules; a hard reference to a service that does not exist yet
  (`'@flag'`) makes the container compile fail and the deploy stop. So every
  service coming from another module — above all one this change is also
  installing — is injected as **optional** with the `?` operator, typed
  nullable, and guarded before use:

  ```yaml
  services:
    Drupal\my_module\Services\BookmarkCounter:
      autowire: true
      arguments:
        $flagService: '@?flag'
  ```

  ```php
  public function __construct(
    private readonly ?FlagServiceInterface $flagService = null,
  ) {}

  public function count(NodeInterface $node): int {
    if ($this->flagService === null) {
      return 0;
    }
    // …
  }
  ```

  The same applies to classes and constants of that module (event names,
  interfaces): code in a module that is not enabled is not autoloadable, so
  `getSubscribedEvents()` returns `class_exists(FlagEvents::class) ? [...] : []`,
  and anything else that touches the module's classes checks
  `ModuleHandlerInterface::moduleExists()` first. The module still goes in
  `.info.yml` `dependencies:` — the optional injection covers the window
  between container rebuild and config import, it does not replace the
  dependency.

## 3. Configuration

- **Schema is mandatory** for every custom config object and every custom
  setting added to an existing one: `config/schema/<module>.schema.yml`, with
  a type for each key (`label` / `text` for translatable strings). No schema,
  no config.
- **Import, then export, then adopt the export.** After writing config YAML
  and importing it (installing the module, `drush cim --partial --source=…`,
  `drush config:set`), export it back (`drush cex`, or
  `drush config:get <name> --format=yaml`) and replace your file with the
  exported version. That is what fixes the two usual defects: keys out of
  alphabetical order, and a missing or incomplete `dependencies:` block. For
  config shipped in `config/install` or `config/optional`, strip `uuid` and
  `_core` from the export.
- **Entity and field config comes complete.** A field needs
  `field.storage.*` + `field.field.*` and, when the entity type has displays,
  the `core.entity_form_display.*` and `core.entity_view_display.*` entries
  with the widget/formatter and its settings — and their `dependencies:`
  list the field config. Verify on the site that the field appears in the
  form and in the display, not just that the import succeeded.
- `config/install` applies only when the module is installed. To change the
  config of an already-installed module, ship a `hook_post_update_NAME()`
  (or use `config/optional` for new optional pieces) — editing
  `config/install` after install changes nothing.
- Config is for deployable settings. Runtime values (last run, cursors,
  counters) go to the State API. Secrets never go into exported config — read
  them from `settings.php` / the environment.

## 4. Drush and the environment

- Run drush through the project wrapper (`ddev drush`, `lando drush`,
  `vendor/bin/drush`). Before any command that writes (`cim`, `cex`, `en`,
  `updb`, `sql-*`, a mutating `php:eval`), confirm the target is the local
  site — `drush status` (URI, DB host) — and never pass a site alias other
  than the local one. A remote alias is never a target.
- `drush cr` after every step that adds or changes services, plugins, hooks,
  routes, theme hooks or config.

## 5. Tests

- Do not write tests unless the plan or the user explicitly asks for them.
  Verification is done with drush (`cr`, `php:eval`, `config:get`) and, when
  the project already has a suite covering the touched area, by running it.

## 6. Access and security

- Every route declares access: `_permission`, `_entity_access`,
  `_custom_access` or `_role`. `_access: 'TRUE'` only for an intentionally
  public endpoint, stated as such in the plan. Custom permissions go in
  `<module>.permissions.yml`, with `restrict access: true` for anything that
  can escalate privileges.
- Entity queries call `->accessCheck(TRUE)`; `accessCheck(FALSE)` only where
  bypassing access is the point (cron, system batch, admin-only reports),
  with a comment saying why.
- Output: user-provided strings never reach `#markup` or Twig `|raw`. Use
  `#plain_text`, `Html::escape()`, `Xss::filter()`, `t()` with `@` / `%`
  placeholders, and Twig's autoescape.
- Database: entity queries, or the Database API with placeholders. Never
  concatenated SQL.
- State-changing requests go through the Form API (CSRF token built in) or a
  route with `_csrf_token: 'TRUE'` / `_csrf_request_header_token: 'TRUE'`.
- Anything exposed to the outside (JSON:API, REST, custom endpoints) gets
  input validation, explicit access, and flood control when it can be
  brute-forced — `get_core_library_doc({library: "Core/Flood"})`.
- Never log or return secrets, tokens, or full request bodies.

## 7. Cacheability

- Cacheability is about **correctness**, not decoration: do not sprinkle
  `#cache`, tags, contexts, lazy builders or cache bins over everything.
  Study each piece of output you produce — what it varies by (user,
  permissions, route, language) and what it is derived from (which entities,
  which config, which custom data) — and attach exactly the contexts and tags
  that make it correct, nothing more.
- Two things are never acceptable: output that varies per user or per
  permission without the matching context, and output derived from an entity
  or config that goes stale after that entity or config changes because its
  tag is missing.
- Anything beyond that (custom cache tags for your own data, `#lazy_builder`
  for one uncacheable fragment, a dedicated cache bin) is a proposal in the
  plan with the case for it — implemented when the user agrees, not by
  default.

## 8. Heavy work — no over-engineering

- Do not introduce queues, batches, cron workers or background processing on
  your own initiative. When the request involves external calls, imports or
  bulk updates, say so in the plan or outline as a recommendation with the
  trade-off (done inline in the request vs. deferred), and implement the
  simple path unless the user opts in — then `Core/Queue` / `Core/Batch`
  docs via `get_core_library_doc`.
- Basic hygiene still applies without being asked: `loadMultiple()` and
  bounded queries (`->range()`) instead of loading inside loops; no
  unbounded queries in hooks that fire on every request.

## 9. Module hygiene

- `.info.yml`: `core_version_requirement`, `package`, and a `dependencies:`
  list naming every module whose API the code uses (`drupal:node`,
  `flag:flag`).
- Prefix everything the module owns with its machine name: permissions,
  routes, config keys, theme hooks, cache tags, queue names, logger channels.
- Logging through an injected `logger.channel.<module>`; messages to users
  through the injected `MessengerInterface`; no `dpm()`, `var_dump()`, or
  `\Drupal::logger()` inside classes.
- All UI strings translatable (`t()`, `StringTranslationTrait`,
  `TranslatableMarkup`) with placeholders — never concatenate translated
  fragments.
- Drupal coding standards: 2-space indentation, docblocks, PSR-4, one class
  per file, typed properties and constructor promotion. Run
  `phpcs --standard=Drupal,DrupalPractice` when the project ships it.
- Theming: no PHP in Twig; theme hooks registered in `hook_theme` with their
  variables; preprocess sets variables, it does not build markup.

## 10. Comments — followed to the letter

Comments follow the Drupal standard — full sentences with a capital letter and
a period; summary lines of at most 80 characters, in the third person
("Returns…", "Builds…"); `@param` / `@return` / `@throws` descriptions on the
next line, indented two spaces; examples between `@code` and `@endcode` — and
these limits, which are not negotiable:

| where | what it says | limit |
|---|---|---|
| inside a method | what a non-obvious step does | 1–2 lines |
| inside a method | a business rule, or why it is done this way | ideal 2 lines, never more than 4 |
| method docblock | summary line, then an optional body | summary 1 line; body at most 3 lines |
| `@param` / `@return` / `@throws` | a short description | exactly 1 line each |
| class docblock | what it is and **how to use it** | as long as it takes, with `@code` examples |

- A method that implements an interface or overrides a parent gets
  `{@inheritdoc}` and nothing else — unless its behaviour differs from the
  contract, and then that difference is the summary.
- No comment restates the code (`// Load the node.` above `$node = …`); no
  commented-out code; no `@todo` without its follow-up (an issue or a plan
  step).
- **No comment narrates the change.** A comment describes the code as it is,
  never what was just done to it: no `// Added null check for the flag
  service`, `// Changed to use the injected service`, `// Updated per the
  plan`, `// Now handles anonymous users`, `// Removed the old query`. The
  reader of the file has no "before" to compare with; the history of an edit
  belongs in the commit message and in the final report, not in the source.
  When editing existing code, the surrounding comments are rewritten to
  describe the new state — never appended with a note about the edit.
- The class docblock is the module's documentation. For a service, a plugin,
  an event, a hook class or a subscriber it says what the class is for, how it
  is obtained or registered, and shows how to use it.

```php
/**
 * Computes the final price of an order from its items and active discounts.
 *
 * Registered by class name: inject it by type, or fetch it with
 * \Drupal::service(OrderPricing::class) from a static context. Discounts come
 * from the my_module.settings config object (see config/schema).
 *
 * @code
 * $total = $this->orderPricing->total($order);
 * @endcode
 */
final class OrderPricing {

  /**
   * Returns the order total after discounts.
   *
   * Discounts apply in the order they are configured; one that would make
   * the total negative is skipped.
   *
   * @param \Drupal\my_module\Entity\OrderInterface $order
   *   The order to price.
   *
   * @return int
   *   The total in cents.
   */
  public function total(OrderInterface $order): int {
    $total = $this->subtotal($order);
    foreach ($this->discounts() as $discount) {
      // Skipped rather than clamped: a negative total would hide a
      // misconfigured discount instead of surfacing it.
      if ($discount > $total) {
        continue;
      }
      $total -= $discount;
    }
    return $total;
  }

}
```

```php
/**
 * Fired after an order is priced and before the total is saved.
 *
 * Subscribe to adjust the total — loyalty points, rounding, surcharges:
 *
 * @code
 * public static function getSubscribedEvents(): array {
 *   return [OrderPricedEvent::NAME => 'onOrderPriced'];
 * }
 *
 * public function onOrderPriced(OrderPricedEvent $event): void {
 *   $event->setTotal($event->getTotal() + 100);
 * }
 * @endcode
 */
final class OrderPricedEvent extends Event {

  public const NAME = 'my_module.order_priced';

}
```

## 11. One-off drush scripts and update hooks

- A drush script (`drush php:eval`, `drush scr`) that creates config, fields,
  displays or content is fine as a **local convenience** — building config
  through the API is often easier than hand-writing YAML. It is never the
  deliverable: nothing a script did on the local site reaches any other
  environment.
- Everything a script produced must therefore land in something that deploys:
  - **Config** → run the script, then export (`drush cex`, or
    `drush config:get <name> --format=yaml`) and ship the YAML: in
    `config/sync` for site config, or in the module's `config/install` /
    `config/optional` (`uuid` and `_core` stripped) when the module owns it.
    Section 3 applies in full.
  - **Changes an already-installed site needs** — migrating data, touching
    existing entities or config, anything a fresh install would not do — go
    in an update hook, and the choice matters because of the order
    `drush deploy` runs them in (`updb` → `cim` → `cr` → `deploy:hook`):
    `hook_update_N()` in `<module>.install` only for schema changes;
    `hook_post_update_NAME()` in `<module>.post_update.php` for changes that
    do **not** depend on config being imported first;
    `hook_deploy_NAME()` in `<module>.deploy.php` for changes that need the
    newly imported config (a new field, a new bundle) to exist. The script
    you ran locally is the prototype of that hook, not a replacement for it.
- Deciding takes judgement: is the site already deployed somewhere? Does the
  project deploy config with `cim`? Does the change touch existing rows or
  only shipped config? Which module should own the hook? When any of that is
  unclear — a brand-new project without a deploy pipeline, no obvious owning
  module, environments you cannot see — **stop and ask the user** before
  choosing. State what the script did locally and the options to make it
  permanent; do not pick one silently.
- The script is not committed unless the user wants it kept. Once its effect
  is in YAML or in a hook, it has done its job.

## 12. Language: everything in English

- Everything generated or implemented is in English, whatever language the
  request came in: identifiers, comments and docblocks, UI strings (the
  source string passed to `t()` — translation is Drupal's job), config
  labels and descriptions, log and exception messages, permission titles,
  file names, the plan file, commit messages when asked to commit.
- The conversation with the user stays in the user's language; the artifacts
  do not.
