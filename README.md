# Pixelite Media — Reusable GitHub Actions Workflows

Shared CI workflows for the WordPress plugin family under
[`pixelitemedia/*`](https://github.com/pixelitemedia). One YAML file
to maintain, ~20 plugins inherit it.

## Available workflows

| Workflow | Trigger | Purpose |
|---|---|---|
| `wp-plugin-ci.yml` | push, PR | Syntax + sniffs + composer validity, informational only |
| `deploy.yml` | caller-controlled | rsync plugin to a known target server (sandbox, trywpem) |
| `wp-org-release.yml` | tag push (caller-controlled) | Deploy a free plugin to wordpress.org SVN |

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

### `deploy.yml` — Deploy plugin to a test/demo server via rsync

Pushes the plugin's working tree to `wp-content/plugins/<slug>/` on a target
server via SSH+rsync, locked down by `rrsync` on the receiving side so the
deploy key can only write into the plugins folder.

**Known targets** (server config baked into the workflow):

| `target` | Host | User | Tier |
|---|---|---|---|
| `sandbox` | `192.254.232.182` (sandbox.em.cm) | `msykes` | bleeding-edge |
| `trywpem` | `162.241.217.222` (trywpem.com)   | `trywpemc` | versioned + manual |

Each target consumes one org secret (`DEPLOY_KEY_EM_CM`, `DEPLOY_KEY_TRYWPEM`)
holding the matching ed25519 private key.

#### Bleeding-edge deploy (push to main)

```yaml
# .github/workflows/deploy-sandbox.yml
name: Deploy to sandbox

on:
  push:
    branches: [main]

jobs:
  sandbox:
    uses: pixelitemedia/workflows/.github/workflows/deploy.yml@v1
    with:
      target: sandbox
    secrets:
      DEPLOY_KEY: ${{ secrets.DEPLOY_KEY_EM_CM }}
```

#### Versioned + manual deploy (releases + on-demand)

```yaml
# .github/workflows/deploy-trywpem.yml
name: Deploy to trywpem

on:
  release:
    types: [published]
  workflow_dispatch:

jobs:
  trywpem:
    uses: pixelitemedia/workflows/.github/workflows/deploy.yml@v1
    with:
      target: trywpem
    secrets:
      DEPLOY_KEY: ${{ secrets.DEPLOY_KEY_TRYWPEM }}
```

#### Inputs

| Input | Default | Purpose |
|---|---|---|
| `target` | _(required)_ | `sandbox` or `trywpem` |
| `plugin-slug` | repo name | Override folder name on the server |
| `rsync-extra-excludes` | _(empty)_ | Newline-separated extra `rsync --exclude` patterns |

Default excludes cover `.git*`, `.github`, `node_modules`, `tests`, `phpunit.xml*`,
`.phpcs.xml*`, `.DS_Store`. Rsync runs with `--delete` so stale files get cleaned up.

### `wp-org-release.yml` — Deploy a free plugin to wordpress.org

Wraps [10up/action-wordpress-plugin-deploy](https://github.com/10up/action-wordpress-plugin-deploy)
behind a thin reusable workflow with a job-summary block on the run page.

#### Use it

Add `.github/workflows/wp-org-release.yml` to your plugin repo:

```yaml
name: Release to wp.org

on:
  push:
    tags:
      - 'v*'

jobs:
  release:
    uses: pixelitemedia/workflows/.github/workflows/wp-org-release.yml@v1
    secrets:
      SVN_USERNAME: ${{ secrets.WP_ORG_USER }}
      SVN_PASSWORD: ${{ secrets.WP_ORG_PASSWORD }}
```

#### Required secrets (per repo)

| Secret | Value |
|---|---|
| `WP_ORG_USER` | wp.org username with commit access to the plugin's SVN |
| `WP_ORG_PASSWORD` | wp.org password (or app password if 2FA enabled) |

Set these as **repository secrets** on each of the four free-plugin repos at
`https://github.com/pixelitemedia/<repo>/settings/secrets/actions`. Free
GitHub plans don't support org-level secrets on private repos, so each
repo holds its own copy.

#### How tags map to wp.org versions

| You push | wp.org receives |
|---|---|
| `git tag v7.2.3.1 && git push origin v7.2.3.1` | wp.org tag `7.2.3.1` (the `v` prefix is stripped by the action) |

The `readme.txt`'s `Stable tag:` header **must match** the git tag. If
the values diverge, the deploy fails.

#### Optional inputs

```yaml
jobs:
  release:
    uses: pixelitemedia/workflows/.github/workflows/wp-org-release.yml@v1
    with:
      slug: 'custom-slug'              # defaults to repo name
      assets-dir: 'wp-org-assets'      # defaults to .wordpress-org
      build-dir: './build'             # path of built plugin if not repo root
    secrets:
      SVN_USERNAME: ${{ secrets.WP_ORG_USER }}
      SVN_PASSWORD: ${{ secrets.WP_ORG_PASSWORD }}
```

## Versioning

- `@v1` — current stable, accepts non-breaking changes
- `@main` — bleeding edge, not recommended
- `@<sha>` — pin exactly, manual updates

When breaking changes ship, they go on `v2` and `v1` keeps working.
