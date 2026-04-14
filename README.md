# Trigger Deploy Action

Triggers the deploy workflow based on detected changes. Handles the logic for determining what type of deploy to trigger (full deploy, k8s-only, config restart) for non-prod environments.

## Usage

```yaml
- name: Trigger deploy
  uses: bisnow/github-actions-trigger-deploy@v1
  with:
    build_container: ${{ needs.check-changes.outputs.build_container }}
    k8s_changed: ${{ needs.check-changes.outputs.k8s_changed }}
    cf_changed: ${{ needs.check-changes.outputs.cf_changed }}
    config_changed_non_prod: ${{ needs.check-changes.outputs.config_changed_non_prod }}
    tag: ${{ env.TAG }}
```

## Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `build_container` | Whether container was built (true/false) | Yes | - |
| `k8s_changed` | Whether k8s files changed (true/false) | Yes | - |
| `cf_changed` | Whether CloudFormation changed (true/false) | Yes | - |
| `config_changed_non_prod` | Whether non-prod ConfigMap/Secret changed (true/false) | Yes | - |
| `tag` | Image tag to deploy | Yes | - |
| `deploy_workflow` | Deploy workflow file name | No | `deploy.yaml` |
| ~~`config_changed_prod`~~ | ~~Whether prod ConfigMap/Secret changed (true/false)~~ | ~~No~~ | ~~Deprecated~~ |

> **Deprecated:** `config_changed_prod` was removed. Prod pod restarts are no longer triggered automatically by this action. If passed, the input is silently ignored.

## Behavior

The action evaluates the change detection outputs and triggers the appropriate deploy:

### Non-prod Deployments

| Condition | Action |
|-----------|--------|
| `build_container=true` | Full deploy with new image tag |
| `k8s_changed=true` or `cf_changed=true` | K8s/CF-only deploy, skips image update |
| `config_changed_non_prod=true` | Adds `restart-pods=true` to pick up config changes |

## Deploy Matrix

| Change Type | Non-prod |
|-------------|----------|
| App code only | New image deployed |
| K8s manifests | K8s sync, no restart |
| ConfigMap/Secret | K8s sync + pod restart |

## Requirements

- The `GH_TOKEN` environment variable must be set (uses `${{ github.token }}` automatically)
- Requires `actions: write` permission in the calling workflow
- Expects a `deploy.yaml` workflow (or custom via `deploy_workflow` input) with these inputs:
  - `tag` - Image tag
  - `environment` - Target environment (non-prod/prod)
  - `skip-image-update` - Boolean to skip image tag update
  - `skip-cloudformation` - Boolean to skip CloudFormation deploy
  - `restart-pods` - Boolean to restart pods after deploy

## Example Workflow

```yaml
name: Build and push

permissions:
  id-token: write
  contents: write
  actions: write

on:
  push:
    branches: [main]

jobs:
  check-changes:
    runs-on: ubuntu-latest
    outputs:
      build_container: ${{ steps.check.outputs.build_container }}
      k8s_changed: ${{ steps.check.outputs.k8s_changed }}
      cf_changed: ${{ steps.check.outputs.cf_changed }}
      config_changed_non_prod: ${{ steps.check.outputs.config_changed_non_prod }}
    steps:
      - name: Check for changes
        id: check
        uses: bisnow/github-actions-check-changes-k8s@v1

  build-and-push:
    needs: check-changes
    if: ${{ needs.check-changes.outputs.build_container == 'true' }}
    # ... build steps ...

  trigger-deploy:
    needs: [check-changes, build-and-push]
    if: ${{ !cancelled() && needs.build-and-push.result != 'failure' }}
    runs-on: ubuntu-latest
    steps:
      - name: Trigger deploy
        uses: bisnow/github-actions-trigger-deploy@v1
        with:
          build_container: ${{ needs.check-changes.outputs.build_container }}
          k8s_changed: ${{ needs.check-changes.outputs.k8s_changed }}
          cf_changed: ${{ needs.check-changes.outputs.cf_changed }}
          config_changed_non_prod: ${{ needs.check-changes.outputs.config_changed_non_prod }}
          tag: rc-${{ github.run_number }}
```

## Related Actions

- [github-actions-check-changes-k8s](https://github.com/bisnow/github-actions-check-changes-k8s) - Detects changes for build, k8s, CloudFormation, and config files

## Versioning

This action uses rolling major version tags. You can pin to:

- A specific version: `@v3.1.0` (exact, never changes)
- A major version: `@v3` (recommended, gets bug fixes and new features)

When a new semantic version tag (e.g., `v3.2.0`) is pushed, a GitHub Actions workflow automatically updates the corresponding major version tag (`v3`) to point to the new release.