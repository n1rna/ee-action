# ee-action

A reusable GitHub Action that uses the [ee CLI](https://github.com/n1rna/ee-cli) to generate environment files from `.ee` config files.

It parses a multi-line `KEY=VALUE` GitHub secret, exports the values as environment variables, and runs `ee hydrate` to produce the output env file.

## Usage

```yaml
- name: Prepare environment - Web
  uses: n1rna/ee-action@v1
  with:
    environment: ${{ inputs.environment }}
    config_path: .ee.web
    env_file: .env.web.${{ inputs.environment }}
    gh_secret: ${{ secrets.ENV_FILE_WEB }}
```

## Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `environment` | Yes | | Target environment name passed to `ee hydrate` |
| `config_path` | Yes | | Path to the `.ee` config file |
| `env_file` | Yes | | Output file path for the generated env file |
| `gh_secret` | Yes | | Multi-line `KEY=VALUE` string from a GitHub secret |
| `format` | No | `dotenv` | Output format (`dotenv`, `json`, `yaml`) |
| `ee_version` | No | `latest` | ee CLI version to download from releases |

## How it works

1. **Installs the ee CLI** from [n1rna/ee-cli](https://github.com/n1rna/ee-cli) GitHub releases, selecting the correct binary for the runner's OS and architecture.
2. **Parses `gh_secret`** line-by-line, exporting each `KEY=VALUE` pair as a shell environment variable (blank lines and `#` comments are skipped).
3. **Runs `ee hydrate`** with the provided environment, config path, output path, and format. The ee CLI resolves schema variables from the exported env vars, falling back to schema defaults.
