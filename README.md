# strict-symfony

A [Claude Code](https://claude.com/claude-code) skill that encodes a strict **PHP 8.5 + Symfony 8** coding standard. The standard is built around positive control flow, no magic literals, Symfony-first solutions and honest audits.

When it is installed, Claude loads it automatically whenever it writes, reviews, tests or audits PHP and Symfony code.

## What it enforces

| ❌ Never | ✅ Instead |
|---|---|
| `!`, `!==`, `!=` | positive conditions; the main path inside `if`, the fallback after it |
| `?:`, `??`, `??=`, `\|\|`, ternaries | `if`, early return, `match` |
| regular expressions | Symfony String, Validator, `Path` |
| `$items[0]`, `$map['key']` | `array_first`, `array_find`, `array_column`, `get()` APIs |
| `sprintf`, `.` concatenation, `append()` chains | `Str::format('Package (package) not found.', [...])` |
| `'--version'`, `'('` literals | `UnicodeString::fromCodePoints(...)` |
| `implode` / `join` / `explode` | `u(\PHP_EOL)->join()`, `u()->split()` |
| literal paths and file names | absolute parameters in `services.yaml` + `#[Autowire(param: ...)]` |
| comments and prose in docblocks | self-explanatory names; tag-only docblocks |
| casts, `@var`, `@phpstan-ignore` to silence PHPStan | real type narrowing; PHPStan level max |
| hand-written helpers | the Symfony component that already does it |

Plus:
- `final readonly` classes and `declare(strict_types=1)`;
- Symfony coding standards;
- tests written under the same rules;
- a precise audit (php -l, PHPStan max, php-cs-fixer `@Symfony`, forbidden-construct grep);
- Homebrew-only tooling and a clean machine afterwards.

## Structure

```
strict-symfony/
├── SKILL.md                    core rules, principles, workflow, reference map
├── references/
│   ├── forbidden.md            every forbidden construct with ❌/✅ examples and edge cases
│   ├── symfony-first.md        which Symfony component replaces which hand-written code
│   ├── patterns.md             canonical shapes: commands, listeners, exceptions, repositories…
│   ├── configuration.md        services.yaml paths, composer.json, PHPStan, PHPUnit
│   ├── testing.md              literal-free tests, sandboxes, MockHttpClient, mutation checks
│   ├── audit.md                exact audit commands, false positives, report template
│   └── workflow.md             scope, cleanup, live systems, git, releases, communication
└── assets/
    ├── Str.php                 the single-pass Str::format utility
    └── phpunit.dist.xml        strict PHPUnit configuration
```

## Install

Personal skill for Claude Code:

```bash
git clone https://github.com/iamsattor/strict-symfony.git ~/.claude/skills/strict-symfony
```

Update:

```bash
git -C ~/.claude/skills/strict-symfony pull
```

Project skill, shared with a team through the repository:

```bash
git clone https://github.com/iamsattor/strict-symfony.git .claude/skills/strict-symfony
```

## License

MIT
