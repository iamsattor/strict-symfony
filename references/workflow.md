# Workflow, environment and collaboration

## Contents

1. [Scope and ownership](#scope-and-ownership)
2. [Where code goes](#where-code-goes)
3. [Asking vs deciding](#asking-vs-deciding)
4. [Facts that change](#facts-that-change)
5. [Tools and installation](#tools-and-installation)
6. [Cleanup](#cleanup)
7. [Working on a live system](#working-on-a-live-system)
8. [Git: commits, pushes, releases](#git-commits-pushes-releases)
9. [Communication](#communication)

---

## Scope and ownership

- **Do exactly what was asked.** "Only the manifest" means only that file: no extra classes, files, options or features. If something nearby is broken, mention it at the end as an offer.
- **The user owns the design.** Keep their names, structure, constraint style (for example `^0.1`) and architecture. Don't argue against a decision. If they ask for your opinion, give an honest recommendation once, then follow their choice.
- **Remove what your change makes dead.** If your change leaves a method unused (such as `Release::latest()` after a refactor), delete it and say so. That is part of a clean change, not scope creep.
- **Stop when told.** If the user rejects a direction mid-way, revert *all* of it (files, parameters, wiring) and confirm nothing remains with grep.

## Where code goes

- Write code into the project files. The chat gets a short summary with file links and **no code blocks**.
- If the user explicitly asks for an example, put it in a file (the scratchpad or the project, as appropriate) and link it.
- Throwaway smoke tests go to the scratchpad and are deleted afterwards.

## Asking vs deciding

**Ask** (a short question, with your recommendation first) when the answer changes what you build and it is the user's call:
- public names (repository, package, command names), architecture (shims, new storage files), behaviour trade-offs (keep old versions or delete them);
- anything that adds shell scripts or non-PHP tooling;
- anything outward-facing, such as publishing, pushing or releasing, unless they already asked for it.

**Decide yourself**, and mention it, when the project already has a convention or there is an obvious default. Examples: following the existing commit style, or putting a parameter next to its siblings.

## Facts that change

Verify on the web before encoding facts that change over time:
- OS releases and names (macOS 27 is "Golden Gate", released 2026-09-14; don't assume the release you know about is the latest);
- package and tool versions (check Packagist or the release page);
- API availability of new components.

## Tools and installation

- Install CLI tools and runtimes **only with Homebrew**: `HOMEBREW_NO_AUTO_UPDATE=1 brew install <formula>`. Never download binaries or phars by hand into temp directories.
- Audit-only tools (php-cs-fixer) are installed for the audit and uninstalled right after it.
- Composer: always pass `--no-cache` (`require`, `update`, `dump-autoload`).
- Project tools (PHPStan, PHPUnit) are dev dependencies and run from `vendor/bin`.

## Cleanup

At the end of every task, remove the junk your work produced, and only that:
- scratchpad files, backups and temporary test roots;
- `var/cache/phpstan`, `var/cache/phpunit` and stale `var/cache/prod`;
- `~/.composer`, `~/Library/Caches/composer`, `~/Library/Caches/Homebrew/downloads/*<formula>*`, `~/Library/Logs/...`.

Check birth times (`stat -f '%SB %N' -t '%H:%M'`) before deleting anything in the home directory. Never touch pre-existing user data.

## Working on a live system

The user may use the tool (for example install packages) while you work on it.
- Snapshot the state files before testing against real state.
- Afterwards restore only *your* changes: remove only the log lines you produced and re-activate what was active before.
- If the state changed in ways you didn't cause, say so and leave it alone.
- If a test went wrong (for example you removed the active version), fix it right away and report it plainly.

## Git: commits, pushes, releases

- Commit or push **only when asked**.
- Follow the repository's existing style. In the user's repositories that means:
  - **one file per commit**;
  - an **emoji-prefixed imperative subject** (`🐛 Replace placeholders in a single pass`, `🧪 Test Str placeholder replacement`, `🛡️ Test Extractor safety checks`);
  - a trailer of `Co-Authored-By: Claude <model> <noreply@anthropic.com>` when the environment provides attribution.
- Commit in logical order: config first, then fixtures, then code and its tests.
- Releases: match the existing tag type (check with `git cat-file -t <tag>`; lightweight vs annotated) and the version scheme (`v0.1.1`). Push the branch and then the tag.
- After a Packagist release, the GitHub webhook usually updates the package within a couple of minutes. Verify with `https://packagist.org/packages/<vendor>/<name>.json` and the p2 metadata before running `composer update` in dependants.
- Public repositories created for the user get an emoji description (`gh repo create <owner>/<name> --public --description "..."`).

## Communication

- Reply in the user's language (Sattor writes in Uzbek).
- Be concise: what changed, what was verified, what remains, and what you'd suggest next.
- Explain trade-offs in plain words when asked "how does X work". Don't use code blocks for that either; describe the flow step by step.
- Never present unverified claims as facts. If something wasn't tested on real hardware (for example Intel or Rosetta paths), say so.
