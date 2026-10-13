# Configuration

## Contents

1. [Paths live in services.yaml](#paths-live-in-servicesyaml)
2. [Application vs library paths](#application-vs-library-paths)
3. [composer.json](#composerjson)
4. [phpstan.dist.neon](#phpstandistneon)
5. [phpunit.dist.xml](#phpunitdistxml)
6. [.gitignore](#gitignore)
7. [Kernel cache](#kernel-cache)

---

## Paths live in services.yaml

Every path, file name and tool location is a parameter with its **full absolute value**:

```yaml
parameters:
    forge.bin: '/opt/forge/bin'
    forge.packages: '/opt/forge/packages'
    forge.cache.partial: '/opt/forge/cache/partial'
    forge.state.requested: '/opt/forge/state/requested.yaml'
    forge.state.history: '/opt/forge/state/history.log'
    forge.sysctl: '/usr/sbin/sysctl'
    forge.open: '/usr/bin/open'
    forge.localtime: '/etc/localtime'
    forge.zoneinfo: '/var/db/timezone/zoneinfo'

services:
    Forge\:
        resource: '/opt/forge/core/src/'
        autowire: true
        autoconfigure: true
```

- **No `%param%` interpolation** (`'%forge.root%/bin'` is forbidden). Each value is spelled out completely.
- **No relative resource paths.** The user dislikes `../src/` and wants `/opt/forge/core/src/`.
- **Naming.** Use dotted lowercase names grouped by area (`forge.state.*`, `forge.cache.*`) and name tools after the tool (`forge.sysctl`).
- **Injection.** `#[Autowire(param: 'forge.state.history')] private string $file`.
- **Explicit services** are listed only when autowiring can't build them: factories (`HttpClient::create`), validator builders, third-party classes.
- **New parameters.** When you add one, keep the list grouped and ordered by area.

## Application vs library paths

| Kind | PSR-4 autoload | Resource paths | Why |
|---|---|---|---|
| Application installed at a fixed prefix (`/opt/forge/core`) | absolute: `"Forge\\": "/opt/forge/core/src/"` | absolute | The user wants full paths. The app only runs from its install location (use a symlink during development). |
| Library published on Packagist | relative: `"Forge\\Archive\\": "src/"`, tests `"tests/"` | relative | It is installed into other projects' `vendor/`, so absolute paths can't work. Report this as a known exception. |

After changing autoload paths, run `composer dump-autoload --no-cache`.

## composer.json

```json
{
    "require": {
        "php": "^8.5",
        "symfony/console": "^8.1"
    },
    "require-dev": {
        "phpstan/phpstan": "^2.3",
        "phpunit/phpunit": "^13.4"
    },
    "autoload": {
        "psr-4": {
            "Forge\\": "/opt/forge/core/src/"
        }
    },
    "autoload-dev": {
        "psr-4": {
            "Forge\\Tests\\": "/opt/forge/core/tests/"
        }
    },
    "config": {
        "sort-packages": true
    },
    "scripts": {
        "check": "phpstan analyse",
        "test": "phpunit"
    }
}
```

- Install with `composer require ... --no-cache`. Never use path repositories or git sources for published packages; take them from Packagist as dist (zip) installs.
- The version constraint style is the user's choice (for example `^0.1`). Don't rewrite it without being asked.
- After editing `require`, update the lock hash with `composer update --lock --no-cache`. Confirm with `composer validate --no-check-publish`.
- Verify current package versions on Packagist before adding them, for example `curl -s https://repo.packagist.org/p2/vendor/name.json`.

## phpstan.dist.neon

```neon
parameters:
    level: max
    paths:
        - config
        - src
        - tests
        - forge
    tmpDir: var/cache/phpstan
```

Don't use `ignoreErrors`, baselines or rule downgrades.

## phpunit.dist.xml

Use `assets/phpunit.dist.xml`. It is strict: risky tests, warnings, deprecations and notices fail the run, output during tests is disallowed, and the cache lives in `var/cache/phpunit`.

## .gitignore

```
/composer.lock
/var/
/vendor/
```

Applications commit `composer.lock`; libraries usually ignore it. Follow what the repository already does.

## Kernel cache

A kernel hard-wired to `prod` with debug off doesn't notice code changes. After editing services, listeners or commands, delete `var/cache/prod` before running the CLI. Delete it again during cleanup, along with `var/cache/phpstan` and `var/cache/phpunit`.
