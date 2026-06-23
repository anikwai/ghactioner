# ghactioner — GitHub Actions cost strategy for Laravel projects

Reference notes and drop-in workflow templates to reduce GitHub Actions spend
across a fleet of Laravel repos.

For Laravel work the cost is rarely macOS minutes. It's structural waste on
cheap Linux runners, multiplied across many repos: workflows that never cancel
superseded runs, re-download every dependency from scratch, and run Xdebug for
coverage nobody reads.

---

## The five cost drivers (and the fix)

| # | Driver | Symptom | Fix |
|---|--------|---------|-----|
| 1 | **No `concurrency` cancel** | Push 5 commits to a PR → 5 full runs bill to completion in parallel | `concurrency` block with `cancel-in-progress: true` |
| 2 | **No dependency caching** | Every run re-resolves Composer **and** re-downloads npm | `actions/cache` for `vendor` + `cache: npm` on `setup-node` |
| 3 | **`coverage: xdebug`** | Tests run 2–5× slower for a coverage report nobody consumes | `coverage: none` (only enable Xdebug/PCOV when you actually publish coverage) |
| 4 | **push + pull_request double-trigger** | The same commit runs CI twice | Trigger on `pull_request` only; reuse the green result on merge (`post-merge.yml`) |
| 5 | **`npm i` / `npm install`** | Non-deterministic, slower installs | `npm ci` against the committed lockfile |

Plus a hard backstop that has nothing to do with YAML:

> **Set a spending limit** in GitHub → Settings → Billing → Actions. Cap it so a
> runaway loop can't silently empty the wallet. Do this first.

The YAML fixes typically cut billed minutes by 50–70% with no change to test
coverage.

---

## Version baseline (keep current; verify before adopting)

| Tool | Use | Why |
|------|-----|-----|
| **Node** | `24` | Active LTS (2026). 22 is maintenance LTS, EOL Apr 2027. |
| **PHP** | `8.5` | Match local and production; `composer.json` `^8.3` allows it. |

Pin both to what your `package.json` `engines` / `composer.json` `require.php`
and your local toolchain actually use. CI should test what you ship.

---

## Templates

Drop-in files live in [`templates/.github/workflows/`](templates/.github/workflows/):

- **`tests.yml`** — Pest/PHPUnit + asset build, PR-only, cached, matrix-named `ci (8.5)`.
- **`lint.yml`** — Pint, static analysis, frontend format/lint/typecheck, cached.
- **`post-merge.yml`** — reuses the green PR checks on the merge commit so CI
  never re-runs on `develop`/`main` (kills the push/PR double-bill).

Copy them into a repo's `.github/workflows/`, then adjust the per-repo knobs
called out in [`templates/README.md`](templates/README.md).

---

## Rollout checklist (per repo)

1. Copy the three templates into `.github/workflows/`.
2. Set `php-version` in the `tests.yml` matrix to the repo's target (e.g. `8.5`).
3. Make `post-merge.yml`'s `requiredChecks` match the actual check names —
   `'ci (<php-version>)'` and `'quality'`.
4. Confirm `composer.lock` and `package-lock.json` are committed (caching + `npm ci` need them).
5. If the repo has docs/security CI steps (e.g. `docs:lint`), do **not** add a
   `paths-ignore` that would skip them.
6. Open a PR and verify: both checks green, cache **hit** on the 2nd run, runtime drops.

Once one repo is proven, repeat across the fleet.
