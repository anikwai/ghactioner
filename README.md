# ghactioner

GitHub Actions workflow templates for Laravel projects, set up to keep CI minutes
low. Comes out of cleaning up CI across a set of Laravel repos that were running
up a much bigger Actions bill than the work justified.

## What was actually costing money

Not macOS minutes. Everything ran on cheap Linux runners. The waste was the same
few mistakes repeated in every repo:

- Workflows had no `concurrency` cancel, so pushing a few commits to a PR left
  several full runs going at once.
- Nothing was cached, so every run downloaded all of Composer and npm from
  scratch.
- Tests ran with `coverage: xdebug`, which is 2-5x slower, for a coverage report
  nobody looked at.
- `tests`/`lint` fired on both `push` and `pull_request`, so the same commit got
  built twice.
- Installs used `npm i` instead of `npm ci`.

Fixing all of that cut billed minutes by roughly half to two thirds without
changing what the tests actually do.

Before touching any YAML, set a spending limit under Settings > Billing >
Actions so a runaway loop can't quietly drain the account.

## Versions

Pin Node and PHP to what you run locally and in production. CI should test what
you ship.

- Node 24 (active LTS). Node 22 is on maintenance and goes EOL April 2027.
- PHP 8.5, which `composer.json: "^8.3"` already allows.

## Templates

The workflows live in `templates/.github/workflows/`:

- `tests.yml` runs Pest/PHPUnit and the asset build. PR-only and cached. Its
  check is named `ci (8.5)` via the matrix.
- `lint.yml` runs Pint and the frontend format/lint checks. PR-only and cached.
- `post-merge.yml` copies the green PR check results onto the merge commit, so
  CI doesn't run again on a push to `develop` or `main`.

Copy them into a repo's `.github/workflows/` and adjust the per-repo settings
described in `templates/README.md`.

## Applying to a repo

1. Copy the three templates into `.github/workflows/`.
2. Set `php-version` in the `tests.yml` matrix to that repo's version.
3. Update `requiredChecks` in `post-merge.yml` to match the check names, which
   are `ci (<php-version>)` and `quality`.
4. Make sure `composer.lock` and `package-lock.json` are committed, since
   caching and `npm ci` rely on them.
5. If the repo has docs or security steps like `docs:lint`, don't add a
   `paths-ignore` that would skip them.
6. Open a PR and check that both jobs pass, the cache hits on the second run,
   and the runtime drops.

Prove it on one repo, then do the rest.
