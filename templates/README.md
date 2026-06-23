# Workflow templates

Drop-in, cost-optimized GitHub Actions workflows for Laravel projects. Copy into
a repo's `.github/workflows/` and adjust the per-repo knobs below.

## Files

| File | Produces check | Purpose |
|------|----------------|---------|
| `tests.yml` | `ci (8.5)` | Composer + npm install (cached), asset build, `php artisan test`. PR-only. |
| `lint.yml` | `quality` | Pint + frontend format/lint (cached). PR-only. |
| `post-merge.yml` | — | Reuses the green PR checks on the merge commit so CI never re-runs on a push to develop/main. |

## Why they're cheap

- **`concurrency` + `cancel-in-progress`** — superseded PR runs are killed, not billed.
- **Composer `vendor` cache + `cache: npm`** — no full re-download every run.
- **`coverage: none`** — no Xdebug tax on test runtime.
- **`pull_request`-only triggers + `post-merge.yml`** — the same commit is never CI'd twice.
- **`npm ci`** — deterministic, faster than `npm i`.

## Per-repo knobs to check before committing

1. **PHP version** — set it in three places consistently:
   - `tests.yml` → `matrix.php-version`
   - `lint.yml` → `setup-php.php-version` and the Composer cache key (`php8.5`)
   - `post-merge.yml` → `requiredChecks` must list `ci (<that version>)`
2. **Branches** — the `on.pull_request.branches` list. Add `workos` etc. if used.
3. **`requiredChecks` names** — must exactly match what your jobs emit
   (`quality`, `ci (8.5)`). If you rename jobs or change the matrix, update this.
4. **Lockfiles committed** — `composer.lock` + `package-lock.json` must exist
   (caching and `npm ci` depend on them).
5. **Extra steps** — if the repo runs `composer analyse`, `npm run types:check`,
   docs/security checks, Wayfinder generation, etc., add them back into `lint.yml`
   / `tests.yml`. The templates are the minimal Laravel starter-kit baseline.
6. **Asset build** — drop the `Build Assets` step from `tests.yml` only if no test
   depends on compiled Vite output.

## Verify after applying

Open a PR and confirm:
- `quality` and `ci (8.5)` both go green.
- The 2nd run shows a **cache hit** (look for "Cache restored from key" in the
  Composer cache step and npm setup step).
- Wall-clock runtime drops noticeably vs. the uncached first run.
- On merge, `post-merge.yml` succeeds and stamps the checks onto the merge commit.
