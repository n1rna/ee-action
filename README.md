# ee-action

Reusable GitHub Actions for the [ee CLI](https://github.com/n1rna/ee-cli) — manage environment variables with schema-based validation.

This repo provides two actions:

| Action | Usage | Description |
|--------|-------|-------------|
| **Hydrate** | `n1rna/ee-action@v1` | Generate env files from GitHub secrets + schema defaults |
| **Push** | `n1rna/ee-action/push@v1` | Push env secrets to GitHub or Cloudflare |

---

## Hydrate (`n1rna/ee-action@v1`)

Generate environment files from a bundled GitHub secret using `ee hydrate`.

### Usage

```yaml
- name: Prepare environment
  uses: n1rna/ee-action@v1
  with:
    environment: production
    config_path: .ee
    env_file: .env.production
    gh_secret: ${{ secrets.ENV_PRODUCTION }}
```

### Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `environment` | Yes | | Target environment name passed to `ee hydrate` |
| `config_path` | Yes | | Path to the `.ee` config file |
| `env_file` | Yes | | Output file path for the generated env file |
| `gh_secret` | Yes | | Multi-line `KEY=VALUE` string from a GitHub secret |
| `format` | No | `dotenv` | Output format (`dotenv`, `json`, `yaml`) |
| `ee_version` | No | `latest` | ee CLI version to download from releases |

### How it works

1. **Installs the ee CLI** from GitHub releases (auto-detects OS and architecture)
2. **Writes `gh_secret`** to a temp file and applies it as environment variables
3. **Runs `ee hydrate`** to resolve schema variables from the applied env vars, falling back to schema defaults

---

## Push (`n1rna/ee-action/push@v1`)

Push environment secrets from `.env` files to GitHub Actions secrets or Cloudflare Workers.

This is useful when you have a repository dedicated to managing `.env` files and want to sync them to remote targets automatically.

### Usage — Push to GitHub

```yaml
- name: Push secrets to GitHub
  uses: n1rna/ee-action/push@v1
  with:
    environment: production
    config_path: .ee
    origin: github
    gh_token: ${{ secrets.GH_PAT }}
```

> **Note:** `GITHUB_TOKEN` cannot create/update secrets in other repos. Use a Personal Access Token (PAT) with `repo` scope, or a fine-grained token with "Secrets" read/write permission.

### Usage — Push to Cloudflare Workers

```yaml
- name: Push secrets to Cloudflare
  uses: n1rna/ee-action/push@v1
  with:
    environment: production
    config_path: .ee
    origin: cloudflare
    cloudflare_api_token: ${{ secrets.CLOUDFLARE_API_TOKEN }}
```

### Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `environment` | Yes | | Target environment name to push |
| `config_path` | No | `.ee` | Path to the `.ee` config file |
| `origin` | No | *(auto)* | Origin name (auto-resolved if only one configured) |
| `mode` | No | *(from config)* | Override push mode (`bundled` or `individual`) |
| `dry_run` | No | `false` | Preview what would be pushed without executing |
| `gh_token` | No | | GitHub token for `gh` CLI (required for GitHub origins) |
| `cloudflare_api_token` | No | | Cloudflare API token for `wrangler` (required for Cloudflare origins) |
| `ee_version` | No | `latest` | ee CLI version to download from releases |

### How it works

1. **Installs the ee CLI** from GitHub releases (auto-detects OS and architecture)
2. **Installs wrangler** if a Cloudflare API token is provided and wrangler is not already available
3. **Runs `ee push`** with the configured origin, environment, and mode — reading `.env` files from the repo and pushing to the remote target

---

## End-to-End Example

A central env repo that pushes secrets to multiple services on commit:

```yaml
# .github/workflows/sync-secrets.yml
name: Sync Secrets
on:
  push:
    branches: [main]

jobs:
  push-secrets:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      # Push API secrets to GitHub
      - name: Push API secrets
        uses: n1rna/ee-action/push@v1
        with:
          environment: production
          config_path: .ee.api
          origin: github
          gh_token: ${{ secrets.GH_PAT }}

      # Push worker secrets to Cloudflare
      - name: Push worker secrets
        uses: n1rna/ee-action/push@v1
        with:
          environment: production
          config_path: .ee.worker
          origin: cloudflare
          cloudflare_api_token: ${{ secrets.CLOUDFLARE_API_TOKEN }}
```

And in a consuming repo, hydrate the secrets at deploy time:

```yaml
# .github/workflows/deploy.yml
steps:
  - uses: actions/checkout@v4

  - name: Prepare environment
    uses: n1rna/ee-action@v1
    with:
      environment: production
      config_path: .ee
      env_file: .env.production
      gh_secret: ${{ secrets.ENV_PRODUCTION }}

  - run: docker build --env-file .env.production -t myapp .
```
