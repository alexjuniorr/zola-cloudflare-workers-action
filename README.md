# Zola → Cloudflare Workers

GitHub Action that checks, builds and (optionally) deploys a [Zola](https://www.getzola.org) site to [Cloudflare Workers](https://developers.cloudflare.com/workers/static-assets/) as static assets.

It runs, in order: `pre-build-command` (optional) → install Zola → `zola check` → `zola build` → `wrangler deploy`.

## Usage

```yaml
name: Deploy

on:
  push:
    branches: [main]
  pull_request:

permissions:
  contents: read

jobs:
  site:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: OWNER/zola-cloudflare-workers-action@v1
        with:
          api-token: ${{ secrets.CLOUDFLARE_API_TOKEN }}
```

> Replace `OWNER/zola-cloudflare-workers-action` with the repository where this action is published.

That is all. By default the action:

- builds and checks the site on every run, including pull requests;
- deploys only on pushes, schedules and manual runs on the default branch (never on pull requests);
- names the Worker after the repository and serves the `public` directory, using Zola's `404.html` for missing pages;
- needs no `wrangler.toml`. If your project has one (`wrangler.toml`, `wrangler.json` or `wrangler.jsonc` in `working-directory`), it is used as is.

## Requirements

1. A Cloudflare API token created from the **Edit Cloudflare Workers** template, stored as the repository secret `CLOUDFLARE_API_TOKEN`.
2. An `ubuntu-latest` runner (x86_64 Linux); Zola is downloaded from the official releases.

The Worker is created on the first deploy; no setup in the Cloudflare dashboard is needed.

## Inputs

| Input | Default | Description |
|---|---|---|
| `api-token` | – | Cloudflare API token. Required to deploy. |
| `account-id` | – | Cloudflare Account ID. Only needed if the token has access to more than one account. |
| `worker-name` | repository name | Worker name. Ignored if the project has its own wrangler config. |
| `deploy` | `auto` | `auto` deploys on the default branch (never on pull requests). `true` / `false` force it. |
| `zola-version` | `0.21.0` | Zola version (without the `v`). |
| `working-directory` | `.` | Directory containing `config.toml` (and, optionally, your wrangler config). |
| `check-links` | `true` | Run `zola check --skip-external-links`. |
| `pre-build-command` | – | Shell command run before the build (lint, tests, data validation). If it fails, the action fails. |

## Outputs

| Output | Description |
|---|---|
| `deployment-url` | Worker URL (empty if nothing was deployed). |

## Examples

**Validate data before publishing**:

```yaml
with:
  pre-build-command: |
    cargo clippy --locked --manifest-path tools/profile-lint/Cargo.toml -- -D warnings
    cargo run --locked --release --manifest-path tools/profile-lint/Cargo.toml
```

If the command needs a toolchain (Rust, Node…), install it in a previous step.

**Weekly rebuild** (useful when content depends on the build date):

```yaml
on:
  schedule:
    - cron: "0 9 * * 1"
```

**Custom domain or other Worker settings** — add a `wrangler.jsonc` to your repository; the action will use it instead of generating one. Point `assets.directory` at `./public`:

```jsonc
{
  "name": "my-site",
  "compatibility_date": "2025-01-01",
  "assets": { "directory": "./public", "not_found_handling": "404-page" },
  "routes": [{ "pattern": "example.com", "custom_domain": true }]
}
```

**Deployment URL in the job**:

```yaml
- id: site
  uses: OWNER/zola-cloudflare-workers-action@v1
  with: { ... }
- run: echo "${{ steps.site.outputs.deployment-url }}"
```

## Notes

- Build and deploy happen in the same job, so what gets published is exactly what was validated.
- Recommended: set `concurrency` in your workflow to avoid two deploys running at once.
- Input values are passed through environment variables, never interpolated into the script.
- Secrets are not available on pull requests from forks; that is fine, since pull requests never deploy with `deploy: auto`.
- Looking for Cloudflare Pages instead? Use the Pages variant of this action.

## Releasing

```bash
git tag v1.0.0 && git tag -f v1 && git push origin v1.0.0 v1 --force
```

Users reference `@v1` (moving tag) or `@v1.0.0` (pinned).

## License

MIT
