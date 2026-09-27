# Structured Deployments for WordPress: deploy workflow

A reusable GitHub Actions workflow that deploys a WordPress site managed by
[desrosj/structured-deployments-for-wp](https://github.com/desrosj/structured-deployments-for-wp).

Each run does three things:

1. **Deploys code.** It rsyncs (`rsync -az --delete`) every plugin and theme that
   the deploy manifest declares with `"source": "monorepo"` from the calling
   repository's `wp-content/plugins/<slug>/` and `wp-content/themes/<slug>/` to
   the server. Each plugin and theme gets its own job, and the jobs run in
   parallel.
2. **Installs the reconciler.** It installs the `structured-deployments` plugin
   at the `github-release` tag the manifest pins. It skips this when that
   version is already installed. Then it network-activates the plugin, along
   with the manifest's `site_plugin` if one is set.
3. **Reconciles.** It syncs the manifest to the server and runs
   `wp structured-deployments reconcile --yes --timings`. The reconcile log is
   streamed to the job log and copied into the step summary.

## What comes from the manifest

The workflow takes nothing site-specific as an input. It reads it all from the
caller's deploy manifest, so the manifest is the only thing that has to change
when the site does:

| What | Where it comes from |
| --- | --- |
| Plugins and themes to rsync | `monorepo` entries in the `plugins` and `themes` maps, after `shared` is merged with the target env (env entries replace shared ones, and a bare `false` drops one) |
| Plugin bootstrap-activated alongside `structured-deployments` | The top-level `site_plugin` key |
| The `structured-deployments` version to install | The `repo` and `tag` of the merged `structured-deployments` plugin entry |
| Secrets forwarded to the reconciler | Every `${VAR}` anywhere in the manifest, plus every `token_env` value |

Adding a secret to the manifest therefore only takes two steps: create the
secret in each deploy environment, and reference it from the manifest. You do
not need to edit the workflow.

## Usage

```yaml
name: Deploy

on:
  push:
    branches:
      - 'main'
      - 'production'
  workflow_dispatch:

permissions: {}

# Concurrency must be set by the caller: groups are repository-wide, so the
# caller's other workflows that write to the same site can share them.
concurrency:
  group: ${{ github.ref == 'refs/heads/production' && 'production-serial' || 'staging-serial' }}
  cancel-in-progress: false
  queue: max

jobs:
  deploy:
    name: Deploy
    uses: desrosj/structured-deployments-for-wp-action/.github/workflows/deploy.yml@<commit-sha> # v1.0.0
    permissions:
      contents: read
    with:
      target-env: ${{ github.ref == 'refs/heads/production' && 'production' || 'staging' }}
      environment: ${{ github.ref == 'refs/heads/production' && 'Production Deployment' || 'Staging Deployment' }}
```

Pin the workflow to a full commit SHA, as in the example above.

### Inputs

| Input | Required | Default | Description |
| --- | --- | --- | --- |
| `target-env` | yes | | Manifest section to deploy, such as `staging` or `production`. Passed to the reconciler as `--env`. |
| `environment` | yes | | GitHub Environment that the deploy jobs run in. The secrets below are read from it. |
| `manifest` | no | `deploy-manifest.json` | Path to the deploy manifest, relative to the repository root. |
| `distignore` | no | `.distignore` | rsync exclude file (`--exclude-from`), relative to the repository root. It is skipped if the file does not exist. |

### Secrets

The workflow reads secrets from the GitHub Environment named by `environment`.
It does not read them from the caller. A caller has no way to pass environment
secrets to a reusable workflow, so the caller does not need a `secrets:` key.

| Secret | Used for |
| --- | --- |
| `DEPLOY_SSH_KEY` | Private key loaded into the SSH agent |
| `DEPLOY_USER` | SSH username |
| `DEPLOY_HOST` | SSH host |
| `DEPLOY_PLUGIN_PATH` | Parent `wp-content/plugins` directory. Each plugin is rsynced into `$DEPLOY_PLUGIN_PATH/<slug>/`. |
| `DEPLOY_THEME_PATH` | Parent `wp-content/themes` directory. Each theme is rsynced into `$DEPLOY_THEME_PATH/<slug>/`. |
| `DEPLOY_WP_PATH` | WordPress root, passed to WP-CLI as `--path` |
| `DEPLOY_MANIFEST_PATH` | Full file path the manifest is synced to and read from on the server |
| `STRUCTURED_DEPLOYMENTS_GITHUB_TOKEN` | Token that can download release assets from the repository in the `structured-deployments` manifest entry |
| *Anything the manifest references* | Every `${VAR}` and `token_env` name. The manifest parser interpolates the whole file, so each one is required in every environment. A missing one fails the reconcile step before it connects to the server. |

The values are spliced into remote shell commands, so use absolute paths
without `~`.

### Jobs

| Job | Environment | Notes |
| --- | --- | --- |
| `plan` | none | Reads the manifest and publishes the deploy plan to the step summary. It never reads secrets. |
| `deploy-code` | `environment` | One matrix job per `monorepo` plugin or theme. Skipped if there are none. |
| `reconcile-apply` | `environment` | Runs only after every `deploy-code` job succeeds (or after `deploy-code` is skipped). |

### Concurrency

Set `concurrency` in the calling workflow, not in this one. Deploys have to
land in push order, because each run rsyncs its own commit with `--delete`,
and the group has to be shared with the other workflows that write to the same
site. Use `cancel-in-progress: false` with `queue: max` so that queued runs
are processed first in, first out, and a deploy is never killed halfway
through a write.
