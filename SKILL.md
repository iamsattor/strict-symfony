---
name: strict-symfony
description: Sattor's strict PHP 8.5 + Symfony 8 coding standard. It is the definition of done for every line of PHP. Use this skill whenever you write, edit, refactor, review, test or audit PHP code, Symfony services, console commands, event listeners, config (services.yaml, composer.json, phpstan or phpunit config) or Composer packages. Also use it when the user mentions their coding rules, toza kod, perfectionist style, an audit, PHPStan max or php-cs-fixer, or asks why code was rejected. It applies even when the request does not mention style, because a one-line PHP fix still has to follow these rules (no negations, no ternaries, no regex, no index access, no sprintf, no literal paths or flags, no comments, Symfony first).
---

# Strict Symfony PHP

This skill describes how Sattor wants PHP to be written. He calls himself a perfectionist. The rules are not preferences to weigh against convenience. They are the acceptance criteria. Code that breaks one of them is unfinished, even when it works.

The rules share one idea. **Code should read as a sequence of positive, typed, Symfony-native statements, with no magic values hidden in it.** Once you understand why each rule exists, you can apply it to cases this file does not list.

## How to use this skill

1. Read this whole file before you write or change any PHP.
2. Then open the reference that matches the task (see the [Reference map](#reference-map)). The references hold the detailed ❌/✅ examples, templates and exact audit commands.
3. Match the surrounding code first. If the project already has a helper (`Str::format`, a `flag()` method, a repository), reuse it. Don't invent a parallel one.
4. Finish with the audit in `references/audit.md` and report the result honestly.

## Core principles (the "why")

| Principle | What it means | Why it matters to the user |
|---|---|---|
| **Symfony first** | Prefer a Symfony component, attribute or helper to hand-written code (String, Filesystem/Path, Finder, Process, Validator, Console, DI attributes, HttpClient, Yaml). | Hand-written helpers are "trash code". They are untested duplicates of solved problems. The codebase should look 100% Symfony. |
| **Positive control flow** | Conditions state what *is* true. The main path lives inside the positive `if`, and the fallback comes after it. | Negations and ternaries hide branches. Positive guards read like sentences and make every exit point visible. |
| **No magic literals** | Paths, file names, CLI flags and delimiter characters never appear as literals in code. | Paths belong in configuration, which is the single source of truth. Flags and delimiters are built explicitly, so nothing gets "stringly" glued together. |
| **Total typing** | PHPStan level max passes with no ignores, no casts and no inline `@var`. | Casts and `@var` hide type bugs instead of fixing them. Narrow types with `\is_*()`, `instanceof` and typed APIs. |
| **Exact scope** | Do what was asked, in the user's shape, naming and architecture. | The user owns the design. Unrequested classes, files or "improvements" are unwelcome, and so is arguing against their decisions. |
| **Honest audits** | Run every check and report the known exceptions openly. | The user trusts the report. A hidden exception or a skipped check is worse than a reported one. |

## The rules at a glance

### Forbidden. Use the alternative instead

| ❌ Forbidden | ✅ Use instead |
|---|---|
| `!`, `!==`, `!=` | Positive checks: `\is_string($x)`, `$x instanceof Y`, `$registry->has($x)`, `null === $x`, set operations (`array_diff`), a boolean helper that returns `false` first and then `true` |
| `? :`, `?:`, `??`, `??=`, `\|\|` | `if` statements, early returns, `match`, `array_any()` |
| Regex (`preg_*`, `#[Assert\Regex]`, regex checks) | Symfony String (`startsWith`, `endsWith`, `containsAny`, `before`, `after`, `split`, `ignoreCase()`), Validator constraints, `Path`, `filter_var` |
| Index access (`$items[0]`, `$map['key']`) | `array_first()`, `array_last()`, `array_find()` (with key), `array_column()`, `array_intersect_key($map, array_flip([$key]))`, `get()`-style APIs |
| `sprintf()`, `.` concatenation, `u()->append()` chains | `Str::format('Package (package) not found.', ['package' => $package])` |
| Quotes around values in messages | `Package composer not found.`, not `Package "composer" not found.` |
| CLI flag literals (`'--version'`) and delimiter literals (`'('`, `':'`) | `UnicodeString::fromCodePoints(45, 45, 118, ...)`, usually through a small `flag(int ...$codePoints)` method |
| `implode()`, `join()`, `explode()` | `u(\PHP_EOL)->join($items)`, `u()->join([...])`, `u($text)->split($separator)` |
| Literal paths and file names (`'bin'`, `'.installed'`) | An absolute parameter in `config/services.yaml`, injected with `#[Autowire(param: '...')]` |
| Commas as list separators in CLI output | One value per line (`$io->listing()`, `writeln($list)`) |
| `version_compare($a, $b, '>=')` | `Composer\Semver\Comparator`, `Semver::satisfies()`, int-backed enums |
| Inline comments (`//`, `#`, `/* */`) and prose in docblocks | Self-explanatory names. Docblocks contain tags only. |
| Casts, `@var`, `@phpstan-ignore` used to silence PHPStan | Real narrowing: `\is_*()`, `instanceof`, typed generics (`@extends Foo<Bar>`) |
| Hand-written helpers, manual validation, boilerplate | The Symfony component that already solves it |
| Shell scripts (`.sh`) and similar non-PHP tooling in the project | PHP using Symfony Process. Big architecture detours need an explicit yes first. |

Every row is explained with examples and edge cases in `references/forbidden.md`.

### Required

- `declare(strict_types=1);` in every PHP file.
- Classes are `final`, and `readonly` where possible. Use enums for closed sets, and value objects instead of arrays-as-structs.
- Reference global classes and constants with a leading backslash (`\Throwable`, `\RuntimeException`, `\PHP_EOL`). Never import them with `use`. Compiler-optimized functions are written as `\count()`, `\is_string()`, `\in_array()` and `\array_key_exists()`, as the `@Symfony` php-cs-fixer ruleset writes them.
- Follow Symfony coding standards: single quotes, short array syntax, trailing commas in multiline lists, braces on their own line for classes and methods.
- Target PHP `^8.5` and use modern natives: `array_first`, `array_last`, `array_find`, `array_any`, `array_all`, first-class callables (`$this->valid(...)`), `new Foo()->bar()` without wrapping parentheses, and named arguments where they help.
- Every non-void method has a docblock with `@return` (and `@param` and `@throws` where they apply). Void methods get a docblock only when they need `@param` or `@throws` tags.
- Domain exceptions use static named constructors that build their messages with `Str::format`.
- Pass the audit: `php -l`, PHPStan max, php-cs-fixer `@Symfony` dry run and the forbidden-construct grep. The tools are installed with Homebrew and removed afterwards.

## Workflow for any coding task

1. **Understand the request and its scope.** Do exactly what was asked. If the request is ambiguous about something that is the user's decision (a name, an architecture, a public API), ask. Otherwise use the project's established convention.
2. **Read the surrounding code.** Copy its idioms: helper names, exception style, docblock density, command structure. `references/patterns.md` shows the canonical shapes.
3. **Write the code into the project files.** Never paste code in chat. The chat gets a short summary with file links.
4. **Prefer Symfony.** Before writing a helper, check `references/symfony-first.md`. If Symfony has it, use it.
5. **Check the literals.** Every path or file name becomes a `services.yaml` parameter. Every flag or delimiter becomes `fromCodePoints`. Every message goes through `Str::format`. See `references/configuration.md`.
6. **Verify.** Run `php -l`, run a throwaway smoke test in the scratchpad (never in the project) and then delete it. Run the test suite if the project has one. Add or adjust tests in the same style (`references/testing.md`).
7. **Audit.** Follow `references/audit.md` exactly. Treat grep hits as leads and classify them as real or as false positives.
8. **Clean up.** Uninstall audit tools, delete the caches and temp files you created (check timestamps) and leave the user's data untouched. See `references/workflow.md`.
9. **Report.** Give a short summary in the user's language, links to the changed files, what was verified, and the known exceptions that remain. Don't include code blocks.

## Known, accepted exceptions

Report these openly in every audit. Never hide them, and never "fix" them by breaking something else.

- The entry script's `require_once __DIR__.'/vendor/autoload.php';`. Nothing is autoloadable before it.
- PHP attribute arguments such as `#[Option(shortcut: 'r')]` or `#[AsCommand(aliases: ['link'])]`. Attributes only accept constant expressions, so `fromCodePoints()` cannot be used there.
- Semantic casts that convert a value instead of silencing PHPStan. For example, `(string) $value` inside `Str::format` stringifies `int|\Stringable`, and an `(int)` cast parses a numeric field from tool output.
- Library packages published to Packagist use relative PSR-4 paths (`src/`, `tests/`), because they are installed under `vendor/`. Applications use absolute paths.
- Absolute tool paths the user approved explicitly, such as a `Tool` enum holding `/usr/bin/tar`.
- A value that is relative by nature (a file name searched for in every project directory) can't be absolute. Raise this with the user before relying on it.

## Collaboration rules (summary)

- **Don't argue against the user's design.** Keep their names, structure and choices. Offer one alternative only when asked or when something is broken.
- **No unrequested architecture changes.** If a feature would need shell scripts, shims or a redesign of core pieces, say so up front and wait for an explicit yes.
- **Verify facts that change on the web** (OS versions and names, latest package versions) before you encode them.
- **Install tools only with Homebrew.** Never download binaries or phars by hand. Prefer no-cache flags such as `composer --no-cache`.
- **Leave the Mac clean.** Delete only what you created, and check timestamps first.
- **Live systems.** Before you test against real state (installed packages, state files), snapshot it. Afterwards restore only your own changes, never the user's.
- **Commit or push only when asked.** Follow the repository's existing commit style. See `references/workflow.md`.

## Reference map

| Read this | When |
|---|---|
| `references/forbidden.md` | Before you write any PHP. It explains every forbidden construct with ❌/✅ examples and edge cases. |
| `references/symfony-first.md` | Before you write any helper or validation, or when you are unsure which component to use. |
| `references/patterns.md` | When you create a class: command, listener, exception, repository, enum, value object, service or renderer. |
| `references/configuration.md` | When you touch paths, `services.yaml`, `composer.json`, `phpstan.dist.neon` or `phpunit.dist.xml`. |
| `references/testing.md` | When you write or change tests, or set up PHPUnit. |
| `references/audit.md` | Before you report a task as done. It has the exact commands and the known false positives. |
| `references/workflow.md` | For the environment, cleanup, commits, releases and the reporting format. |
| `assets/Str.php` | When a project has no `Str::format` utility yet. Copy it to `src/Support/Str.php` and adapt the namespace. |
| `assets/phpunit.dist.xml` | When you set up PHPUnit in a project. |
