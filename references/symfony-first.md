# Symfony first: component catalog

Before writing any helper, check this list. If a component covers the need, use it. Hand-written duplicates of these are "trash code".

## Contents

1. [Strings — symfony/string](#strings--symfonystring)
2. [Files and paths — symfony/filesystem, symfony/finder](#files-and-paths)
3. [External tools — symfony/process](#external-tools--symfonyprocess)
4. [Validation — symfony/validator](#validation--symfonyvalidator)
5. [Console — symfony/console](#console--symfonyconsole)
6. [Dependency injection — symfony/dependency-injection](#dependency-injection)
7. [Events — symfony/event-dispatcher](#events--symfonyevent-dispatcher)
8. [HTTP — symfony/http-client](#http--symfonyhttp-client)
9. [Data files — symfony/yaml](#data-files--symfonyyaml)
10. [Versions — composer/semver](#versions--composersemver)
11. [Modern PHP natives](#modern-php-natives)

---

## Strings — symfony/string

Use `u($text)` (`function Symfony\Component\String\u;`) and `UnicodeString`.

| Need | API |
|---|---|
| a delimiter or flag character | `UnicodeString::fromCodePoints(45, 45)` |
| join | `u(\PHP_EOL)->join($items)`, `u()->join([...])` |
| split | `u($s)->split($separator)`, `->split($separator, 2)` |
| before or after a needle | `->before($needle)`, `->after($needle)` (both return the whole string when the needle is missing, so check `containsAny` first when that matters) |
| starts, ends, contains | `->startsWith()`, `->endsWith()`, `->containsAny()` |
| case | `->ignoreCase()`, `->lower()` |
| trim and pad | `->trim()`, `->trimEnd()`, `->padEnd($width)` |
| ensure a prefix | `->ensureStart($prefix)` |
| length and slice | `->length()`, `->slice($start, $length)` |
| equality | `->equalsTo($other)` |
| messages | the project's `Str::format()` (see `assets/Str.php`) |

## Files and paths

**symfony/filesystem.**

| Need | API |
|---|---|
| read or write | `$filesystem->readFile($f)`, `->dumpFile($f, $content)`, `->appendToFile($f, $content)` |
| exists, remove, mkdir | `->exists()`, `->remove($pathOrList)`, `->mkdir()` |
| copy, mirror, rename | `->copy()`, `->mirror($from, $to, options: ['override' => true])`, `->rename()` |
| symlinks | `->symlink($target, $link)`, `->readlink($link)`, `->readlink($path, true)` (canonical, `null` when missing) |
| permissions | `->chmod($path, 0o755)` |
| temporary files | `->tempnam($dir, $prefix)` |

`dumpFile()` **follows symlinks** and writes to the target. Remove a link first if you mean to replace the link itself.

**`Path`** (`Symfony\Component\Filesystem\Path`): `join`, `makeAbsolute`, `makeRelative`, `isBasePath`, `getDirectory`, `canonicalize`. Use `Path::isBasePath($root, $candidate)` for every "inside the directory?" security check.

**symfony/finder.** `Finder::create()->files()->ignoreDotFiles(false)->ignoreVCS(false)->depth(0)->sortByName()->in($dir)`. Use `iterator_to_array($finder, false)` to get a list, `$file->getPathname()` and `$file->getRelativePathname()` for names, and `->hasResults()` to test for emptiness.

## External tools — symfony/process

- Always pass an array command: `new Process([$this->tool, $this->flag(45, 110), $key])`. Never use a shell string.
- Tool paths come from parameters (or from a user-approved enum).
- For long operations (archives, git, installs) pass `timeout: null`.
- Use `->mustRun()` when failure is exceptional (catch `ProcessFailedException` at the boundary), and `->run()` plus `0 === $exit` when failure is an expected answer.
- Put `--` before operands that come from users or files, so a name like `--use-compress-program=…` can't become an option.
- Interactive passthrough: `if (stream_isatty(\STDOUT)) { $process->setTty(Process::isTtySupported()); }`. Otherwise stream the output with a `run()` callback and keep `Process::ERR` separate.
- Find a command on PATH with `ExecutableFinder::find($name)`, which returns `?string` and must be narrowed.

## Validation — symfony/validator

- Put attributes on value objects: `#[Assert\NotBlank]`, `#[Assert\Url(protocols: ['https'], requireTld: true)]`, `#[Assert\Length(exactly: 64)]`, `#[Assert\Type('xdigit')]`, `#[Assert\Count(min: 1)]`, `#[Assert\Unique(fields: [...])]`, `#[Assert\All(constraints: [...])]`, `#[Assert\Valid]`.
- Validate untrusted input (YAML or JSON files) with `new Assert\Sequentially(constraints: [...])` so validation stops at the first failure.
- Throw `new ValidationFailedException($value, $violations)` when `$violations->count() > 0`. Don't create a custom exception for validation.
- For user-facing messages, map `$exception->getViolations()` to `(path) (message)` lines with `Str::format`, then trim them.
- Build a validator outside the container with `Validation::createValidatorBuilder()->enableAttributeMapping()->getValidator()`.

## Console — symfony/console

- **Invokable commands.** Use a `final readonly class` with `#[AsCommand(name: ..., description: ...)]` and `__invoke(SymfonyStyle $io, OutputInterface $output, #[Argument] ..., #[Option] ...)`. Return `Command::SUCCESS` or `Command::FAILURE`.
- **Arguments.** `#[Argument(description: '...')] string $name`. An array argument is `array $packages` (required) or `array $packages = []` (optional). Never call an argument `command`, because that name is reserved.
- **Options.** `#[Option(description: '...', shortcut: 'j')] bool $json = false`. `?string $release = null` makes a value option.
- **SymfonyStyle output.**
  - Messages: `success`, `error`, `warning`, `note`.
  - Structure: `section`, `listing`, `definitionList`, `table`.
  - Questions: `confirm('Do you want to continue?')`. It returns the default in non-interactive (`-n`) mode, like `apt -y`.
  - Errors in JSON mode go to `$io->getErrorStyle()`.
- **Trees.** `TreeHelper::createTree($io, new TreeNode($root, $children))->render()`.
- **Progress.** Use `ConsoleSectionOutput` per item and `ProgressBar` inside a section. Call `$section->clear()` before the first bar draw so the item stays on one line. Format sizes with `Helper::formatMemory($bytes)`.
- **Aliases.** `#[AsCommand(name: 'use', aliases: ['link'])]` instead of a duplicate command.
- **Listeners.** Use `#[AsEventListener]` on an invokable for `ConsoleCommandEvent`. Call `$event->disableCommand()` to block a command and `ConsoleSignalEvent` to handle interrupts.
- **Raw output.** Use `$output->writeln($text, OutputInterface::OUTPUT_RAW)` for machine output such as JSON or shell lines.

## Dependency injection

- Register `services.yaml` resources with `autowire: true` and `autoconfigure: true`. Exclude classes with `#[Exclude]` (for example renderers built manually).
- Inject parameters with `#[Autowire(param: 'forge.bin')] private string $bin` and other services with `#[Autowire(service: 'console.command_loader')]`.
- Tag collections:
  - `#[AutoconfigureTag('forge.manifest')]` on the base class;
  - `#[AsTaggedItem('composer')]` on each implementation;
  - `#[AutowireLocator('forge.manifest')] private ServiceLocator $manifests` to receive them.
- Make a service public with `#[Autoconfigure(public: true)]` only when the entry point needs to fetch it.
- Kernels use `AbstractKernel` + `KernelTrait` with bundles listed in `config/bundles.php`. The compiled container is cached, so clear the cache (`var/cache/prod`) after code changes when the kernel runs without debug mode.

## Events — symfony/event-dispatcher

- Domain events are `final class XEvent extends Event` with `public readonly` properties.
- Listeners are `final readonly class` invokables with `#[AsEventListener]` and a single typed `__invoke(XEvent $event): void`.
- Use one listener per concern (history, checksums) instead of one listener with many branches.
- Use `EventSubscriberInterface` only for objects created manually (renderers), registered with `$dispatcher->addSubscriber()`.

## HTTP — symfony/http-client

- Inject `HttpClientInterface`, created with `HttpClient::create()` through a services.yaml factory.
- Stream several downloads with `$client->stream($responses)` and handle `ChunkInterface` (`isFirst()`, `getContent()`).
- Resume downloads with `'headers' => ['Range' => Str::format('bytes=(offset)-', [...])]` and handle `206`, `200` and `416`.
- Report progress with the `on_progress` callback.
- Tests use `MockHttpClient` + `MockResponse($body, ['http_code' => 206])`. Never touch the network.

## Data files — symfony/yaml

- `Yaml::parseFile($file)` returns `mixed`. Narrow it (`\is_array`, `array_filter(..., \is_string(...))`) or validate it with Validator.
- `Yaml::dump($data)` writes it back. State files that become empty are removed instead of being written as `[]`.

## Versions — composer/semver

- `Semver::satisfies($version, $constraint)` and `Comparator::greaterThan($a, $b)` for comparisons.
- `VersionParser::normalize()` inside `try`/`catch (\UnexpectedValueException)` to detect versions.
- Sorting: `usort($artifacts, static fn (Artifact $a, Artifact $b): int => version_compare($b->version, $a->version))`. The two-argument form is fine.

## Modern PHP natives

Use these instead of loops and index access:

| Function | Typical use |
|---|---|
| `array_first()` / `array_last()` | the first or last element, narrowed afterwards |
| `array_find($a, fn ($value, $key))` | the first match, or a map value by key |
| `array_any()` / `array_all()` | quantifiers |
| `array_column($objects, 'prop')` | project public properties |
| `array_combine($keys, $values)` | build maps |
| `array_replace($map, [$k => $v])` | set a key immutably |
| `array_diff_key($map, array_flip([$k]))` | remove a key |
| `array_merge(...$lists)` | flatten a list of lists (an empty spread returns `[]`) |
| `iterator_to_array($it, false)` | iterator to list |
| `strtr($text, $pairs)` | single-pass multi replacement (inside `Str::format`) |
| `filter_var($v, \FILTER_VALIDATE_INT)` | integer parsing |
| first-class callables | `$this->repository->installed(...)`, `\is_string(...)` |
| `new Foo()->bar()` | PHP 8.4+ syntax, no extra parentheses |
