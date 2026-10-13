# Testing

Tests follow exactly the same rules as production code: no negations, no ternaries, no index access, no literal paths, no comments, no casts and no `@var` overrides. PHPStan max analyses `tests/` as well.

## Contents

1. [Setup](#setup)
2. [Filesystem test case](#filesystem-test-case)
3. [Literal-free test data](#literal-free-test-data)
4. [Sandboxes over mocks](#sandboxes-over-mocks)
5. [Typed fixtures for PHPStan](#typed-fixtures-for-phpstan)
6. [Assertions and exceptions](#assertions-and-exceptions)
7. [Data providers](#data-providers)
8. [HTTP, console and processes](#http-console-and-processes)
9. [Hostile input without shell scripts](#hostile-input-without-shell-scripts)
10. [Proving a test protects something](#proving-a-test-protects-something)
11. [Live systems](#live-systems)

---

## Setup

1. Run `composer require --dev "phpunit/phpunit:^13.4" --no-cache`. Check the current version on Packagist first.
2. Copy `assets/phpunit.dist.xml` to the project root.
3. Add an `autoload-dev` entry (`Forge\Tests\` → absolute `tests/` for apps, relative for libraries) and a `"test": "phpunit"` script. Then run `composer dump-autoload --no-cache`.
4. Add `tests` to the `phpstan.dist.neon` paths.
5. Mirror `src/` in `tests/` (`tests/Installer/LinkerTest.php` tests `src/Installer/Linker.php`). Shared helpers go in `tests/Fixture/`.

## Filesystem test case

Every test that touches the disk extends an abstract base. The base creates an isolated temporary root and removes it afterwards:

```php
abstract class FilesystemTestCase extends TestCase
{
    protected Filesystem $filesystem;

    private string $root;

    protected function setUp(): void
    {
        $this->filesystem = new Filesystem();
        $this->root = $this->real(Path::join(sys_get_temp_dir(), $this->random()));
    }

    protected function tearDown(): void
    {
        $this->filesystem->remove($this->root);
    }

    /**
     * @return string
     */
    protected function random(): string
    {
        return bin2hex(random_bytes(8));
    }

    /**
     * @return string
     */
    protected function directory(): string
    {
        return $this->real(Path::join($this->root, $this->random()));
    }

    /**
     * @return string
     */
    protected function file(): string
    {
        return Path::join($this->root, $this->random());
    }

    /**
     * @return string
     *
     * @throws \LogicException
     */
    private function real(string $directory): string
    {
        $this->filesystem->mkdir($directory);

        $path = $this->filesystem->readlink($directory, true);

        if (\is_string($path)) {
            return $path;
        }

        throw new \LogicException('The directory is not available.');
    }
}
```

- **Canonical paths.** On macOS `sys_get_temp_dir()` lies under `/var`, which is a symlink to `/private/var`. Code that compares canonical paths (`readlink(..., true)`) fails unless the root itself is canonical, and `real()` handles that.
- **`TestCase::name()` is final.** Name the helper `random()`.
- **Overrides.** Never override a base helper with a different meaning (for example redefining `file()` in a subclass). Add a new private method instead.

## Literal-free test data

- File and directory names come from `random()`, never from `'tool'`, `'bin'` and the like.
- Real configuration values (tool paths) are read from the booted kernel instead of being repeated:
  ```php
  /**
   * @return string
   *
   * @throws \LogicException
   */
  protected function parameter(string $name): string
  {
      $kernel = new Kernel();
      $kernel->boot();

      $value = $kernel->getContainer()->getParameter($name);

      if (\is_string($value)) {
          return $value;
      }

      throw new \LogicException('The parameter is not a string.');
  }
  ```
- A helper that always fails comes from PATH: `new ExecutableFinder()->find('false')`, narrowed with `\is_string`.
- Domain data such as package names, versions and manifest binary maps is plain data. Use class constants (`private const string BINARY = 'tool';`) when a value is repeated.

## Sandboxes over mocks

Prefer the **real object graph** in temporary directories to mocks. A `Sandbox` fixture builds registry, repositories, linker and resolver from real classes, and offers helpers such as `install()` and `activate()`. Final readonly classes can't be mocked anyway. Real wiring catches integration bugs that mocks would hide.

For manifests and plugins, write a configurable test implementation (`TestManifest` with binaries, artifacts, dependencies, conflicts, commands and a `broken` flag whose `verify()` throws).

## Typed fixtures for PHPStan

When a generic can't be inferred (`new ServiceLocator([...])` gives `ServiceLocator<mixed>`), write a tiny typed fixture instead of using `@var`:

```php
/**
 * @extends ServiceLocator<Manifest>
 */
final class ManifestLocator extends ServiceLocator
{
    /**
     * @param array<string, Manifest> $manifests
     */
    public function __construct(array $manifests)
    {
        parent::__construct(array_map(static fn (Manifest $manifest): \Closure => static fn (): Manifest => $manifest, $manifests));
    }
}
```

## Assertions and exceptions

- Use `$this->assertSame` for exact values, and `assertEquals` / `assertContainsEquals` for value objects.
- Use `$this->expectExceptionObject(ResolverException::packageNotFound('missing'))` to check class, message and code through the domain's own named constructor.
- To assert something **after** the exception (for example that nothing was written), use `try` / `$this->fail(...)` / `catch` and then the assertions:
  ```php
  try {
      $this->downloader(new MockResponse('tampered'))->download($this->installation);

      $this->fail('The checksum mismatch was not detected.');
  } catch (InstallerException $exception) {
      $this->assertSame('Checksum mismatch for release 1.0.0.', $exception->getMessage());
  }

  $this->assertFileDoesNotExist($this->download());
  ```
- Join output lines with `u(\PHP_EOL)->join(...)` and assert with `assertStringContainsString` / `assertStringEndsWith`. Don't index into the result.
- Turn console output into lines with Symfony String: `u($display)->split(\PHP_EOL)` mapped through `->trimEnd()->toString()`.

## Data providers

```php
/**
 * @return iterable<string, array{MacOS, MacOS, bool}>
 */
public static function systems(): iterable
{
    yield 'same release' => [MacOS::Tahoe, MacOS::Tahoe, true];
    yield 'older release' => [MacOS::Sonoma, MacOS::Sequoia, false];
}

#[DataProvider('systems')]
public function testIsAtLeast(MacOS $system, MacOS $minimum, bool $expected): void
```

Use attributes (`#[DataProvider]`) and named `yield` keys.

## HTTP, console and processes

- **HTTP:** `new MockHttpClient([new MockResponse('ary', ['http_code' => 206])])`. Cover success, resume, a skipped (already verified) download, checksum mismatch and HTTP errors.
- **Events:** use a real `EventDispatcher` with a listener that appends to a typed property (`$this->events = [...$this->events, ...]`).
- **Commands:** `new CommandTester(new VersionsCommand(...))`. It accepts invokable commands directly. Use `$tester->execute(['package' => 'lib', '--json' => true])`, `assertCommandIsSuccessful()` and `getStatusCode()`.
- **Console sections and TUI:** a `BufferedConsoleOutput extends StreamOutput implements ConsoleOutputInterface` (stream from `tmpfile()`) that exposes its `ConsoleSectionOutput`s, so tests can count lines per section.
- **Listeners:** build the event (`new ConsoleCommandEvent(new Command('list'), new ArrayInput([]), new BufferedOutput())`), invoke the listener and assert `commandShouldRun()`.

## Hostile input without shell scripts

Build malicious archives from PHP with the real tool and an mtree specification instead of committing binaries or shell scripts:

```php
protected function tar(string ...$operands): string
{
    $archive = $this->file();

    new Process([Tool::Tar->value, $this->flag(45, 80), $this->flag(45, 99, 102), $archive, ...$operands], $this->root)->mustRun();

    return $archive;
}
```

An mtree line such as `Str::format('(name) type=link link=(target)', [...])` produces `../escape` entries, symlinks outside the destination, fifos or `mode=0000` files. Compute traversal names with `Path::makeRelative()` instead of writing `'../'`.

## Proving a test protects something

For every safety check or bug fix:
1. Write the test.
2. Confirm it **fails** without the fix: temporarily remove the check (back the file up to the scratchpad), run the filtered suite, then restore the file and confirm `git status` is clean.
3. Confirm it passes with the fix.

Report which checks were mutation-verified.

## Live systems

When a command has to be exercised against real installed state:
- Snapshot the state files first (copy them to the scratchpad).
- Undo exactly what you did afterwards. For example, reactivate the previously active release *before* pruning, so you never remove the user's active version.
- Never delete the user's packages, history lines or state written outside your test window. Remove only your own entries.
