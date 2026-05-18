# Pixelite Media — Reusable GitHub Actions Workflows

Shared CI workflows for the WordPress plugin family under
[`pixelitemedia/*`](https://github.com/pixelitemedia). One YAML file
to maintain, ~20 plugins inherit it.

## Available workflows

### `wp-plugin-ci.yml` — Standard WordPress plugin CI

Four jobs, all **informational only** (`continue-on-error: true`) — they
never block merges. The goal is to surface noise floor across legacy
plugins before deciding what to harden.

| Job | What it does | Blocking? |
|---|---|---|
| `syntax` | `php -l` across PHP 7.4 → 8.3 matrix | No |
| `phpcs` | PHPCS against `WordPress-Core` standard with PR annotations | No |
| `php-compatibility` | PHPCompatibilityWP, auto-detects `Requires PHP` from plugin header | No |
| `composer-validate` | Validates `composer.json` if present | No |

Findings show up three ways:

1. **PR-line annotations** via `cs2pr` (when run on a pull_request event)
2. **Job summary tables** on the workflow run page: finding counts and
   top sniff sources for both PHPCS and PHP compatibility
3. **Checkstyle XML artifacts** (`phpcs-checkstyle`, `phpcompat-checkstyle`)
   downloadable from the run page for full per-line detail

Individual sniff steps now exit with their real status (yellow/red on
failures), but the job-level `continue-on-error: true` keeps the overall
workflow `success` so merges aren't gated.

#### Use it

Add `.github/workflows/ci.yml` to your plugin repo:

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  ci:
    uses: pixelitemedia/workflows/.github/workflows/wp-plugin-ci.yml@v1
```

#### Customize it

All inputs have sensible defaults; override only what you need:

```yaml
jobs:
  ci:
    uses: pixelitemedia/workflows/.github/workflows/wp-plugin-ci.yml@v1
    with:
      php-versions: '["8.1", "8.2", "8.3"]'   # drop 7.4 / 8.0 if plugin requires 8.1+
      phpcs-standard: 'WordPress'              # heavier: adds Docs + Extra sniffs
      phpcs-paths: 'src includes'              # scan only specific dirs
      phpcs-exclude: '*/vendor/*,*/tests/*'    # extra ignores
```

## Versioning

- `@v1` — current stable, accepts non-breaking changes
- `@main` — bleeding edge, not recommended
- `@<sha>` — pin exactly, manual updates

When breaking changes ship, they go on `v2` and `v1` keeps working.
