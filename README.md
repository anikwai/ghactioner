# ghactioner

Optimized GitHub Actions workflow templates for Laravel apps, tuned to use fewer
CI minutes and keep the Actions bill down.

## The optimizations

- Cancel superseded runs with a `concurrency` block, so new commits to a PR stop
  the old run instead of running both.
- Cache Composer and npm so each run doesn't reinstall everything from scratch.
- Run tests with `coverage: none` (Xdebug is much slower and the coverage report
  wasn't used).
- Trigger on `pull_request` only, and reuse the green result on merge via
  `post-merge.yml`, instead of building the same commit on both push and PR.
- Use `npm ci` instead of `npm i`.

Also set a spending limit under Settings > Billing > Actions as a safety net.

## Templates

In `templates/.github/workflows/`:

- `tests.yml` runs Pest/PHPUnit and the asset build. Check name: `ci (8.5)`.
- `lint.yml` runs Pint and the frontend lint/format checks.
- `post-merge.yml` reuses the PR's passing checks on the merge commit.

Both use Node 24 and PHP 8.5. Pin these to whatever your repo actually runs.

## Promoting develop to main

CI runs on PRs into develop. To promote to production without re-running it:

```
git checkout main && git fetch origin develop
git merge --ff-only origin/develop && git push origin main
git checkout develop
```

The fast-forward makes main point at the exact commit that already passed CI, so
the green checks carry over and no build runs on the push. This needs main to
allow direct pushes (no "require pull request" protection); adding that rule
forces a develop -> main PR, which costs one extra build per release.

## Applying to a repo

1. Copy the templates into `.github/workflows/`.
2. Set `php-version` in `tests.yml` to your version, and match it in
   `post-merge.yml`'s `requiredChecks` (`ci (<php-version>)` and `quality`).
3. Make sure `composer.lock` and `package-lock.json` are committed.
4. Open a PR and confirm both jobs pass and the cache hits on the second run.
