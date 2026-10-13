# Audit

Run the audit before you report any coding task as done. Run every step and report every result. "Honest" means three things: no skipped checks, no hidden exceptions and no claims you didn't verify.

## Contents

1. [Order of checks](#order-of-checks)
2. [Syntax](#1-syntax)
3. [PHPStan max](#2-phpstan-max)
4. [Tests](#3-tests)
5. [php-cs-fixer @Symfony](#4-php-cs-fixer-symfony)
6. [Forbidden-construct grep](#5-forbidden-construct-grep)
7. [Runtime smoke check](#6-runtime-smoke-check)
8. [Cleanup](#7-cleanup)
9. [Report template](#report-template)

---

## Order of checks

1. `php -l` on every changed file.
2. PHPStan level max.
3. The test suite, if there is one.
4. php-cs-fixer `@Symfony` dry run.
5. Grep for forbidden constructs, with each hit classified.
6. A runtime smoke check of the real CLI or service.
7. Cleanup.

## 1. Syntax

```bash
for f in src/Path/One.php src/Path/Two.php; do php -l "$f" | grep -v '^No syntax'; done
```

## 2. PHPStan max

```bash
rm -rf var/cache/prod && vendor/bin/phpstan analyse --no-progress
```

- PHPStan is a dev dependency (`phpstan/phpstan`). Make sure `tests` is in its paths.
- Fix the cause of every error (see `forbidden.md` §12). Never add ignores, `@var` overrides, casts or baselines.

## 3. Tests

```bash
vendor/bin/phpunit
```

For bug fixes and safety checks, also run the mutation check described in `testing.md`.

## 4. php-cs-fixer @Symfony

php-cs-fixer is **not** a project dependency. Install it with Homebrew for the audit and uninstall it afterwards. It refuses several paths without a config file, so loop over the files. `phpdoc_to_comment` and `no_superfluous_phpdoc_tags` are disabled so that the required `@param`/`@return` tags survive:

```bash
HOMEBREW_NO_AUTO_UPDATE=1 brew install php-cs-fixer >/dev/null 2>&1
for f in $(git ls-files -m -o --exclude-standard '*.php'); do
  php-cs-fixer fix --dry-run --diff --using-cache=no --allow-unsupported-php-version=yes \
    --rules='{"@Symfony":true,"phpdoc_to_comment":false,"no_superfluous_phpdoc_tags":false}' -- "$f" \
    | grep -E '^[+-]|Found [1-9]'
done
brew uninstall php-cs-fixer >/dev/null 2>&1
brew cleanup php-cs-fixer >/dev/null 2>&1
find ~/Library/Caches/Homebrew/downloads -name '*php-cs-fixer*' -delete
```

To apply fixes instead of only reporting them, drop `--dry-run --diff`. Then run the dry run again to confirm it is clean. Typical findings are import order, unused imports and docblock alignment.

In a project that isn't a git repository, list the changed files explicitly instead of using `git ls-files`.

## 5. Forbidden-construct grep

Run these against the changed files (`$F` holds the list; in zsh use `${=F}`, or a quoted array in bash). **Every hit must be classified** as a real violation (fix it) or a known false positive (say so).

```bash
grep -nE '!|\?\?|\?:| \? |\|\|' $F | grep -vE '^\S+:[0-9]+:\s*\* @|\?string|\?[A-Z][A-Za-z]+ \$'
grep -nE 'sprintf|implode\(|[^:>]join\(|explode\(|preg_|version_compare|append\(' $F
grep -nE '^\s*//|^\s*#[^\[]|/\*[^*]' $F
grep -nE '@var|@phpstan-ignore|\((int|string|bool|array|float)\)' $F
grep -nE '(\$[A-Za-z_][A-Za-z0-9_]*|\))(->[A-Za-z_]+)*\[' $F
grep -nE "'--?[a-z]|'[()\[\]{}:@#,]'|'/[a-z]" $F
grep -nE '\?->' $F
```

**Known false positives (classify them, don't "fix" them):**

| Hit | Why it is fine |
|---|---|
| `Path::join(` | a path API, not `join()` |
| `?string`, `?Installation $x` | nullable types, not ternaries |
| `/** @var list<State> */` on a property | a property type tag, not an inline override |
| `.'` at the end of a message such as `'Package installed.'` | sentence punctuation, not concatenation |
| `['package' => $package]` | an array literal, not index access |
| `version_compare($b->version, $a->version)` | two arguments in a sort callback, no operator string |
| `(string) $value` in `Str::format` | a documented semantic cast that must be reported |
| `'!'` inside a message | a string literal, not an operator |

## 6. Runtime smoke check

Run the real entry point after clearing the container cache. For a CLI, run the commands you touched plus `list`. When the smoke test needs scaffolding, write it to the scratchpad (never into the project), run it and delete it.

## 7. Cleanup

```bash
rm -rf var/cache/phpstan var/cache/phpunit
for d in ~/.composer ~/Library/Caches/composer; do [ -e "$d" ] && stat -f '%SB %N' -t '%H:%M' "$d"; done
```

- Remove `~/.composer` / `~/Library/Caches/composer` only when their birth time falls inside your work window. Running `composer` and `composer.phar` creates them.
- Remove scratchpad files, Homebrew download leftovers and temporary test roots you created.
- Leave everything else alone.

## Report template

Reply in the user's language, keep it short and use no code blocks:

1. **What changed:** each file as a clickable link, with one line on why.
2. **Checks:** php -l, PHPStan max (error count), tests (count and status), php-cs-fixer (clean, or what was fixed), grep (clean, or the false positives named), runtime check.
3. **Known exceptions that remain:** for example the entry script `require_once`, attribute literals, semantic casts and relative library autoload.
4. **Cleanup:** what was removed.
5. **Open points:** anything you noticed but didn't change because it was out of scope, as an offer, not as an action already taken.
