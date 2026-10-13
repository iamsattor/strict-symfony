# Canonical class shapes

Copy these shapes when you create a new class. They come from real, audited code and follow every rule. Always check the target project first; its existing shapes win.

## Contents

1. [File skeleton](#file-skeleton)
2. [Docblock conventions](#docblock-conventions)
3. [Domain exception](#domain-exception)
4. [Enum with behaviour](#enum-with-behaviour)
5. [Value object with validation](#value-object-with-validation)
6. [Service with injected parameters](#service-with-injected-parameters)
7. [Repository over a state file](#repository-over-a-state-file)
8. [Abstract base with configured subclasses](#abstract-base-with-configured-subclasses)
9. [Console command](#console-command)
10. [Command with confirmation](#command-with-confirmation)
11. [Event listener](#event-listener)
12. [Running external tools](#running-external-tools)
13. [Positive-only control flow recipes](#positive-only-control-flow-recipes)
14. [Collections without loops or indexes](#collections-without-loops-or-indexes)

---

## File skeleton

```php
<?php

declare(strict_types=1);

namespace Forge\Installer;

use Forge\Manifest\Installation;
use Symfony\Component\Filesystem\Filesystem;

use function Symfony\Component\String\u;

final readonly class Example
{
    public function __construct(
        private Filesystem $filesystem,
    ) {
    }
}
```

- Order: `declare`, namespace, class imports, a blank line, then function imports.
- Never import global classes; write `\RuntimeException`, `\Throwable` and `\PHP_EOL` with a leading backslash.
- Use promoted constructor properties with trailing commas. The empty constructor body is `{` on the next line followed by `}`.

## Docblock conventions

- Every method with a non-`void` return type has `@return`, even when the type is obvious (`@return bool`, `@return string`).
- Add `@param` only when the type is richer than the signature (`list<string>`, `array<string, string>`, `non-empty-string`, `int<0, max>`).
- List every exception the method can let escape with `@throws`.
- Don't write prose. Don't write `@return void`.
- Properties with rich types get `/** @var list<State> */`.

```php
/**
 * @param list<string> $packages
 *
 * @return list<Installation>
 *
 * @throws ResolverException
 */
public function stale(array $packages): array
```

## Domain exception

```php
abstract class ForgeException extends \RuntimeException
{
}
```

```php
final class ResolverException extends ForgeException
{
    /**
     * @return self
     */
    public static function packageNotFound(string $package): self
    {
        return new self(Str::format('Package (package) not found.', ['package' => $package]));
    }

    /**
     * @return self
     */
    public static function unsupportedRelease(string $release, MacOS $system): self
    {
        return new self(Str::format('Release (release) requires macOS (system) or later.', ['release' => $release, 'system' => $system->value]));
    }
}
```

- Use one exception class per module and one named constructor per situation.
- Messages are full sentences, values are never quoted, and the sentence ends with a period.
- Wrap lower-level failures, keeping `previous`: `new self(Str::format(...), 0, $previous)`.
- Libraries have their own abstract base (`ArchiveException`) so callers can catch everything from the package.
- Commands catch the domain base exceptions plus the expected infrastructure ones (`ProcessFailedException`, `TransportExceptionInterface`) and print `$io->error($exception->getMessage())`.

## Enum with behaviour

```php
enum MacOS: int
{
    case BigSur = 11;
    case Monterey = 12;
    case Tahoe = 26;
    case GoldenGate = 27;

    /**
     * @return bool
     */
    public function isAtLeast(self $system): bool
    {
        return $this->value >= $system->value;
    }
}
```

- Use int-backed enums for ordered things (versions) and string-backed enums for formats and states.
- Map between enums with `match`, never by string value (`Format::from($other->value)` couples two enums silently).
- Verify facts that change on the web, such as the latest OS release or names, before adding cases.

## Value object with validation

```php
final readonly class Artifact
{
    /**
     * @param non-empty-string $version
     * @param non-empty-string $url
     * @param non-empty-string $sha256
     */
    public function __construct(
        #[Assert\NotBlank]
        public string $version,
        #[Assert\NotBlank]
        #[Assert\Url(protocols: ['https'], requireTld: true)]
        public string $url,
        public ArtifactFormat $format,
        public MacOS $minimalOperationSystem,
        #[Assert\Length(exactly: 64)]
        #[Assert\Type('xdigit')]
        public string $sha256,
    ) {
    }

    /**
     * @return bool
     */
    public function supports(MacOS $system): bool
    {
        return $system->isAtLeast($this->minimalOperationSystem);
    }
}
```

## Service with injected parameters

```php
final readonly class Layout
{
    public function __construct(
        #[Autowire(param: 'forge.bin')]
        private string $bin,
        #[Autowire(param: 'forge.packages')]
        private string $packages,
    ) {
    }

    /**
     * @return string
     */
    public function prefix(string $package, string $version): string
    {
        return Path::join($this->packages, $package, $version);
    }
}
```

## Repository over a state file

```php
final readonly class RequestedRepository extends MarkRepository
{
    public function __construct(
        Filesystem $filesystem,
        #[Autowire(param: 'forge.state.requested')]
        string $file,
    ) {
        parent::__construct($filesystem, $file);
    }
}
```

The base holds the behaviour. It reads a YAML list, narrows it with `array_filter(..., \is_string(...))`, writes it back with `Yaml::dump` and **removes the file when it becomes empty**:

```php
/**
 * @param list<string> $packages
 */
private function save(array $packages): void
{
    if (\count($packages) > 0) {
        $this->filesystem->dumpFile($this->file, Yaml::dump($packages));

        return;
    }

    $this->filesystem->remove($this->file);
}
```

## Abstract base with configured subclasses

Share behaviour through an `abstract readonly class` and keep a `final readonly` subclass per configured instance (as above). Don't copy a class and change one parameter.

## Console command

```php
#[AsCommand(name: 'which', description: 'Show which package provides a command')]
final readonly class WhichCommand
{
    public function __construct(
        private InstalledRepository $repository,
        private Layout $layout,
        private Filesystem $filesystem,
    ) {
    }

    /**
     * @return int
     */
    public function __invoke(
        SymfonyStyle $io,
        #[Argument(description: 'The command name')]
        string $name,
    ): int {
        $link = Path::join($this->layout->bin(), $name);

        if (Path::isBasePath($this->layout->bin(), $link)) {
            return $this->render($io, $name, $this->filesystem->readlink($link, true));
        }

        return $this->missing($io, $name);
    }
}
```

- Keep `__invoke` short and move branches into private methods that return `int`.
- Report one error message through one method (`missing()`). Don't copy the same `$io->error()` into several branches.
- A JSON variant goes behind `#[Option(description: 'Output as JSON', shortcut: 'j')] bool $json = false` and a `JsonRenderer`.
- An empty result is a `warning` (or a `success` when "nothing to do" is the good outcome) and returns `Command::SUCCESS`.

## Command with confirmation

```php
/**
 * @param list<Installation> $installations
 *
 * @return int
 *
 * @throws \Throwable
 */
private function uninstall(SymfonyStyle $io, array $installations): int
{
    $io->section('Uninstalling');
    $io->listing(array_map(static fn (Installation $installation): string => $installation->reference(), $installations));

    if ($io->confirm('Do you want to continue?')) {
        $this->installer->uninstall(...$installations);

        $io->success('Packages uninstalled.');

        return Command::SUCCESS;
    }

    $io->warning('Aborted.');

    return Command::FAILURE;
}
```

## Event listener

```php
#[AsEventListener]
final readonly class ChecksumListener
{
    public function __construct(
        private ChecksumRepository $checksums,
    ) {
    }

    public function __invoke(StateChangedEvent $event): void
    {
        match ($event->state) {
            State::Installed => $this->checksums->record($event->installation),
            State::Removed => $this->checksums->remove($event->installation),
            default => null,
        };
    }
}
```

## Running external tools

```php
/**
 * @return bool
 */
private function supported(): bool
{
    $process = new Process([$this->sysctl, $this->flag(45, 110), 'hw.optional.arm64']);

    if (0 === $process->run()) {
        return u($process->getOutput())->trim()->equalsTo('1');
    }

    return false;
}
```

## Positive-only control flow recipes

| Situation | Recipe |
|---|---|
| "if not exists, throw" | `if (exists) { return main; } throw ...;` |
| "unless supported, stop" | `if (supported) { return; } stop();` |
| a predicate negated in a filter | a helper that returns `false` for the positive case and then `true` |
| "items not in set B" | `array_diff`, `array_diff_key`, then filter with a positive `\in_array` |
| a nullable lookup with a fallback | `$x = array_find(...); if ($x instanceof X) { return $x; } return fallback/throw;` |
| a value that may be `false` (`getenv`, `getcwd`, `hash_file`) | `if (\is_string($value)) { ... }` and the fallback after it |
| a default parameter value | a separate method, never `??` |
| recursion until a fixed point | an early-return recursive method (see below) |

```php
/**
 * @param list<string> $packages
 * @param list<string> $needed
 *
 * @return list<string>
 */
private function needed(array $packages, array $needed): array
{
    $pending = array_values(array_unique(array_diff(array_filter($packages, $this->registry->has(...)), $needed)));

    if (\count($pending) > 0) {
        return $this->needed(
            array_merge(...array_map(fn (string $package): array => array_keys($this->registry->get($package)->dependencies()->packages), $pending)),
            [...$needed, ...$pending],
        );
    }

    return $needed;
}
```

## Collections without loops or indexes

- `foreach` is fine for side effects (writing files, validating each item by calling something that throws). Use `array_map` and `array_filter` to transform data.
- Use `array_values()` after `array_filter` / `array_diff` whenever the result is typed as `list<...>`.
- Build maps with `array_combine($keys, $values)`. Read map values with `array_find($map, fn ($value, $key) => ...)` or `array_intersect_key($map, array_flip([$key]))`.
- Flatten with `array_merge(...array_map(...))`.
- Grow state inside closures with `$this->events = [...$this->events, $event];`.
