# Forbidden constructs

Each section gives the rule, the reason behind it, the alternatives and a ❌/✅ pair taken from real code. The ✅ examples follow every other rule in this file too.

## Contents

1. [Negated conditions](#1-negated-conditions)
2. [Ternary and short-circuit operators](#2-ternary-and-short-circuit-operators)
3. [Regular expressions](#3-regular-expressions)
4. [Direct index access](#4-direct-index-access)
5. [Manual string building](#5-manual-string-building)
6. [CLI flag and delimiter literals](#6-cli-flag-and-delimiter-literals)
7. [implode / join / explode](#7-implode--join--explode)
8. [Literal paths and file names](#8-literal-paths-and-file-names)
9. [Commas in CLI output](#9-commas-in-cli-output)
10. [version_compare with an operator](#10-version_compare-with-an-operator)
11. [Comments](#11-comments)
12. [Casts, @var and PHPStan ignores](#12-casts-var-and-phpstan-ignores)
13. [Trash code](#13-trash-code)
14. [Shell scripts and architecture detours](#14-shell-scripts-and-architecture-detours)
15. [Constructs that are allowed](#15-constructs-that-are-allowed)

---

## 1. Negated conditions

**Rule.** Don't use `!`, `!==` or `!=`. Write the positive condition, put the main path inside the `if` and the fallback after it.

**Why.** A negation makes the reader invert the logic in their head. With a positive guard every branch says what *is* true, and a method reads top to bottom as "if this holds, do that; otherwise this".

❌
```php
if (!$this->filesystem->exists($file)) {
    throw ArchiveNotFoundException::fromFile($file);
}

return $this->read($file);
```

✅
```php
if ($this->filesystem->exists($file)) {
    return $this->read($file);
}

throw ArchiveNotFoundException::fromFile($file);
```

**Techniques for writing it positively.**

- **Boolean helper.** When you need "not X" as a callback, write a helper that returns `false` for the positive case first:
  ```php
  /**
   * @return bool
   */
  private function missing(Installation $installation): bool
  {
      if ($this->repository->installed($installation)) {
          return false;
      }

      return true;
  }
  ```
  Then use it as `array_filter($installations, $this->missing(...))`.
- **Set operations.** Use `array_diff`, `array_diff_key` and `array_intersect` instead of filtering with `!in_array`:
  ```php
  $stale = array_diff(array_column($installations, 'prefix'), array_column($kept, 'prefix'));

  return array_values(array_filter($installations, static fn (Installation $installation): bool => \in_array($installation->prefix, $stale, true)));
  ```
- **Equality is positive.** `null === $x`, `$a === $b` and `0 === $process->run()` are allowed.
- **Swap the branches.** For "if this is not X, do Y", check for X, return early, and do Y afterwards:
  ```php
  if ($this->supported()) {
      return;
  }

  $io->error('Forge requires a Mac with Apple silicon (M1 or later).');
  $event->disableCommand();
  ```
- **Negative quantifiers.** "None of them match" is written as `array_all(... positive ...)`, or as `array_any()` with the branches swapped.

**Edge cases.** A `!` inside a string literal is fine; only the operator is forbidden. `!` inside `#[...]` attribute expressions is also not allowed. Pick a positive attribute instead.

---

## 2. Ternary and short-circuit operators

**Rule.** Don't use `? :`, `?:`, `??`, `??=` or `||`. Use `if`, early `return` and `match`.

**Why.** These operators compress branching into one expression, so the fallback path is easy to miss. `if` and `match` give every case its own visible line.

❌
```php
$name = $artifact?->version ?? 'unknown';
$state = $active ? 'active' : 'installed';
```

✅
```php
/**
 * @return string
 */
private function status(Installation $installation): string
{
    if ($this->repository->active($installation)) {
        return 'active';
    }

    return 'installed';
}
```

✅ Use `match` to map closed sets:
```php
return match ($this) {
    self::Phar, self::Binary => null,
    self::Zip => Format::Zip,
    self::TarGz => Format::TarGz,
    self::TarXz => Format::TarXz,
    self::TarBz2 => Format::TarBz2,
};
```

✅ Use `match` for side effects on enum states too:
```php
match ($event->state) {
    State::Installed => $this->checksums->record($event->installation),
    State::Removed => $this->checksums->remove($event->installation),
    default => null,
};
```

**Defaults without `??`.** Look the value up with `array_find()` or `get()`, narrow it with `instanceof` or `\is_string()`, and return the fallback after the `if`.

**`&&`.** It isn't on the forbidden list, but prefer nested positive `if`s, early returns or `array_all()` when they read better. Never combine `&&` with negations.

**`?->` (nullsafe).** It isn't listed either, but it hides a null branch the same way `??` does. Prefer explicit narrowing (`$x instanceof Y`) in new code.

---

## 3. Regular expressions

**Rule.** No `preg_*`, no `#[Assert\Regex]` and no regex-based checks of any kind.

**Why.** Regexes are opaque, easy to get subtly wrong (ReDoS, anchoring, encoding) and nearly always replaceable by a named operation.

| Need | Use |
|---|---|
| prefix or suffix | `u($s)->startsWith($p)`, `u($file)->ignoreCase()->endsWith($extension)` |
| contains | `u($s)->containsAny($needle)` |
| split or tokenize | `u($s)->split($separator)`, `u($s)->split($separator, 2)` |
| before or after a delimiter | `u($line)->before($space)`, `u($line)->after($keyword)` |
| trim or pad | `->trim()`, `->trimEnd()`, `->padEnd($width)` |
| case-insensitive work | `->ignoreCase()`, `->lower()` |
| validation | Validator constraints: `Type`, `NotBlank`, `Url`, `Length`, `Choice`, `Count`, `All`, `Sequentially`, `Unique` |
| integer parsing | `filter_var($value, \FILTER_VALIDATE_INT)` followed by `\is_int()` |
| path checks | `Path::isBasePath()`, `Path::makeRelative()`, `Path::canonicalize()` |
| version parsing | `Composer\Semver\VersionParser::normalize()` inside `try`/`catch` |

---

## 4. Direct index access

**Rule.** Don't read with `$items[0]`, `$map['key']`, `$_SERVER['X']` or list destructuring (`[$a, $b] = ...`).

**Why.** Index access assumes the key exists, and PHPStan max can't trust it. The named functions either narrow the type or return `null`, and you then handle that explicitly.

| Need | Use |
|---|---|
| first or last element | `array_first($list)`, `array_last($list)`, then narrow with `instanceof` or `\is_string` |
| element matching a predicate | `array_find($list, static fn (Node $node): bool => null === $node->type)` |
| value by key in a map | `array_find($map, static fn (string $release, string $name): bool => $name === $package)` (the callback receives value and key) |
| several values by key | `array_values(array_intersect_key($map, array_flip([$package])))` |
| a column of objects or arrays | `array_column($installations, 'prefix')` |
| does the key exist | `\array_key_exists($name, $binaries)` |
| does any element match | `array_any()` / `array_all()` |
| container, locator or config values | `$locator->get($name)`, `$container->getParameter($name)` |

**Building arrays** is fine. Use literals, spread (`[...$a, ...$b]`), `array_map`, `array_combine` and `array_replace($map, [$key => $value])`. To grow a list inside a closure, reassign it: `$this->events = [...$this->events, $event];`.

Example: a nullable lookup followed by narrowing.
```php
$installation = array_first($this->repository->installations($package));

if ($installation instanceof Installation) {
    $this->linker->link($manifest, $installation);

    return;
}

$this->filesystem->remove($this->layout->package($package));
```

---

## 5. Manual string building

**Rule.** Don't use `sprintf()`, the `.` concatenation operator, interpolated strings (`"Hello $name"`) or `u(...)->append(...)` chains. Build every message with the project utility `Str::format()` and its `(name)` placeholders. Never put quotes around values in messages.

**Why.** The templates stay readable and greppable, every value goes through one tested, single-pass function, and messages look the same everywhere.

❌
```php
throw new \RuntimeException(sprintf('Package "%s" not found.', $package));
$label = $name.' '.$version;
```

✅
```php
return new self(Str::format('Package (package) not found.', ['package' => $package]));

return Str::format('(package) (release)', ['package' => $this->package, 'release' => $this->artifact->version]);
```

**`Str::format` must be single-pass.** It replaces all placeholders at once (`strtr` with an array), so a value that contains `(other)` is never substituted again. A project without the utility can copy `assets/Str.php`.

**Joining pieces without a template.** When there is no surrounding text, join the pieces: `u()->join([$open, $key, $close])->toString()`.

**Templates may contain** spaces, `=`, `$PATH` and other plain text. The template is the readable part. Only computed values are passed in as parameters.

---

## 6. CLI flag and delimiter literals

**Rule.** Never write CLI flags (`'--version'`, `'-cf'`) or special delimiter characters (`'('`, `')'`, `':'`, `'@'`, `'.'`) as string literals. Build them with `UnicodeString::fromCodePoints(...)`.

**Why.** Flags and delimiters are syntax, not data. Building them explicitly makes every command line visible and reviewable, and prevents "stringly" glued commands.

✅ The usual shape is a private helper in each class that needs it:
```php
/**
 * @return string
 */
private function flag(int ...$codePoints): string
{
    return UnicodeString::fromCodePoints(...$codePoints)->toString();
}
```

It is used like this:
```php
new Process([Tool::Tar->value, $this->flag(45, 120, 102), $archive, $this->flag(45, 67), $staging], timeout: null)->mustRun();
```

Common code points:

| Char | Code | Char | Code | Char | Code |
|---|---|---|---|---|---|
| space | 32 | `#` | 35 | `(` | 40 |
| `)` | 41 | `,` | 44 | `-` | 45 |
| `.` | 46 | `/` | 47 | `:` | 58 |
| `=` | 61 | `@` | 64 | `!` | 33 |
| `a`–`z` | 97–122 | `A`–`Z` | 65–90 | `0`–`9` | 48–57 |

So `--version` is `fromCodePoints(45, 45, 118, 101, 114, 115, 105, 111, 110)`, `-n` is `fromCodePoints(45, 110)` and `--` is `fromCodePoints(45, 45)`.

**Separators in code** follow the same rule: `UnicodeString::fromCodePoints(32)` for a space, `fromCodePoints(46)` for a dot and `fromCodePoints(58)` for the `PATH` separator.

**Exception.** PHP attribute arguments (`#[Option(shortcut: 'r')]`, `#[AsCommand(aliases: ['link'])]`) must be constant expressions. Report them as known exceptions.

**Plain identifiers are fine.** A sysctl key such as `'hw.optional.arm64'`, a hash algorithm name such as `'sha256'` or a Process name are data, not flags.

---

## 7. implode / join / explode

**Rule.** Don't call the native `implode()`, `join()` or `explode()`.

**Why.** Symfony String handles Unicode correctly and keeps the code within one string API.

✅
```php
u(\PHP_EOL)->join($lines)->toString();
u($output)->split(\PHP_EOL);
u()->join([$first, $second])->toString();
```

Note: `Path::join()` is a path API, not string joining, and it is allowed. The audit grep flags it as a false positive.

---

## 8. Literal paths and file names

**Rule.** Code never contains path segments or file names (`'bin'`, `'packages'`, `'.installed'`, `'/usr/bin/open'`). Declare each one as a parameter in `config/services.yaml` with its **full absolute value**, inject it with `#[Autowire(param: '...')]`, and only join dynamic parts in code.

**Why.** Every location is in one reviewable place. Moving the installation is a config change, and nothing is hard-coded twice.

❌
```php
$bin = Path::join($this->root, 'bin');
new Process(['/usr/sbin/sysctl', '-n', 'hw.optional.arm64']);
```

✅
```yaml
parameters:
    forge.bin: '/opt/forge/bin'
    forge.sysctl: '/usr/sbin/sysctl'
```
```php
public function __construct(
    #[Autowire(param: 'forge.sysctl')]
    private string $sysctl,
) {
}
```

**Interpolation is not allowed in parameters.** `'%forge.root%/bin'` is forbidden. Write `'/opt/forge/bin'`.

**Dynamic parts are fine.** `Path::join($this->packages, $package, $version)` joins configured roots with runtime values.

**In tests.** Use random names (`bin2hex(random_bytes(8))`) for temporary files and directories. Read real parameters from the booted container. See `testing.md`.

**Exceptions to raise with the user:** a value that is relative by nature (a per-project file name), relative autoload paths in published libraries, and absolute tool paths in an enum when the user has approved them.

---

## 9. Commas in CLI output

**Rule.** Never print lists with commas, such as `Installed: a, b, c`. Show one value per line.

**Why.** The output becomes scannable, diffable and easy to grep.

✅
```php
$io->listing(array_map(static fn (Installation $installation): string => $installation->reference(), $installations));
$output->writeln($entries);
```

✅ Inside a table cell, join the values with `u(\PHP_EOL)->join($versions)`.

---

## 10. version_compare with an operator

**Rule.** Don't use `version_compare($a, $b, '>=')` or any other operator string.

**Why.** Operator strings are a stringly-typed mini language. Use typed comparisons instead.

✅
```php
Comparator::greaterThan($latest->artifact->version, $installation->artifact->version);
Semver::satisfies($version, $constraint);
$system->value >= $minimum->value;
```

A two-argument `version_compare($b, $a)` used as a sort callback is allowed. It returns an int and needs no operator string.

---

## 11. Comments

**Rule.** No `//`, `#` or `/* */` comments, and no prose in docblocks. Docblocks contain tags only: `@param`, `@return`, `@throws`, `@var` (on properties), `@implements`, `@extends` and `@template`.

**Why.** Names should explain the code. Comments go stale. Tags are machine-checked contracts.

✅
```php
/**
 * @param list<string> $packages
 *
 * @return list<Installation>
 *
 * @throws ResolverException
 */
public function upgrades(array $packages): array
```

Property type tags are fine: `/** @var list<State> */`. Inline `@var` used to override an inferred type is not. See section 12.

---

## 12. Casts, @var and PHPStan ignores

**Rule.** PHPStan level max must pass with no `@phpstan-ignore`, no baseline, no inline `/** @var X $y */` override and no cast added only to silence an error.

**Why.** Each of these hides a real type problem.

**Narrow types with:**
- `\is_string()`, `\is_int()`, `\is_array()` and `instanceof` checks before use;
- `array_filter($items, \is_string(...))`, which PHPStan understands as narrowing;
- `filter_var(..., \FILTER_VALIDATE_INT)` plus `\is_int()`;
- typed generics on small fixtures, for example `/** @extends ServiceLocator<Manifest> */ final class ManifestLocator extends ServiceLocator`;
- an explicit `throw new \LogicException(...)` when a value is impossible (`getcwd()` returning `false`).

**Semantic casts** are not silencers. Examples are `(string) $value` in `Str::format` for `int|\Stringable`, and `(int)` when parsing a numeric field from tool output. They are allowed but must be listed in the audit report.

---

## 13. Trash code

**Rule.** Don't write helpers, validation or boilerplate when Symfony (or PHP itself) already provides a solution.

**Why.** Every hand-written helper is untested surface area that duplicates a maintained component.

Examples (see `symfony-first.md` for the full catalog):
- manual `if`/`throw` validation → Validator constraints plus `ValidationFailedException`;
- a hand-rolled tree printer → `TreeHelper` / `TreeNode`;
- a home-made HTTP stub in tests → `MockHttpClient` / `MockResponse`;
- `exec()`, `shell_exec()` or backticks → `Process` with array arguments;
- a manual `realpath()` and string-prefix check → `Path::isBasePath()`;
- a manual byte formatter → `Helper::formatMemory()`.

When Symfony has nothing suitable (for example multi-key single-pass replacement), a native PHP function (`strtr`) is the next best choice. Explain this in the report.

---

## 14. Shell scripts and architecture detours

**Rule.** Don't add `.sh` files or other non-PHP tooling, and don't restructure core pieces (link layout, shims, storage format) without an explicit go-ahead.

**Why.** The user wants a pure PHP/Symfony codebase. An unrequested detour was once stopped mid-way and fully reverted.

Do this instead: use `Process` with absolute tool paths from parameters. If a feature really needs a new architecture, explain the trade-off and wait for a clear yes.

---

## 15. Constructs that are allowed

So you don't over-correct:
- `null === $x`, `$a === $b`, `0 === $process->run()` and `\count($x) > 0` are positive comparisons.
- `&&` and `instanceof` are allowed (keep the conditions positive).
- `match (true)` is acceptable but rarely needed. Prefer `match` on an enum or value.
- Spread, `array_map`, `array_filter`, `array_combine`, `array_merge(...$lists)` and first-class callables.
- Named arguments: `new Process($command, timeout: null)`, `new Extractor(limit: 1)`.
- `Path::join()` for dynamic parts.
- String literals that are data: package names, messages, hash algorithm names, sysctl keys.
- Single-argument natives that have no Symfony equivalent: `basename()`, `hash_file()`, `bin2hex()`, `random_bytes()`, `stream_isatty()`, `getcwd()` (narrowed).
