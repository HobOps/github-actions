# github-actions

Shared GitHub Actions for the HobOps organization: reusable workflows and
composite actions.

## Reusable workflows

### `terraform-module-ci.yml`

Full CI/release pipeline for Terraform module repositories
(e.g. `terraform-kubernetes-bootstrap`, `terraform-kubernetes-monitoring`).

| Job | When | Purpose |
|-----|------|---------|
| `preflight` | PR | Branch name (`feat/*`, `fix/*`, `revert-*`) + conventional commits (commitlint) |
| `terraform-lint` | PR + push to main | `terraform fmt`, `validate` (root + `examples/*`), `tflint` |
| `security` | PR + push to main | Checkov scan of the Terraform |
| `release` | push to main | Compute next semver tag from conventional commits and publish a GitHub Release |

The caller repo keeps its own config files: `.tflint.hcl`, `.checkov.yaml`
and `commitlint.config.mjs` (commitlint-github-action v6 requires `.mjs`).

Usage (`.github/workflows/ci.yml` in the caller repo):

```yaml
name: CI

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

concurrency:
  group: ci-${{ github.event.pull_request.number || github.ref }}
  cancel-in-progress: true

permissions:
  contents: read

jobs:
  ci:
    uses: HobOps/github-actions/.github/workflows/terraform-module-ci.yml@v1
    permissions:
      contents: write # release job pushes tags / creates releases
      pull-requests: read # commitlint lists the PR commits
```

Inputs (all optional):

| Input | Default | Description |
|-------|---------|-------------|
| `terraform_version` | `1.9.8` | Terraform version for fmt/validate |
| `tflint_version` | `latest` | TFLint version |
| `checkov_config_file` | `.checkov.yaml` | Checkov config in the caller repo |
| `commitlint_config_file` | `commitlint.config.mjs` | commitlint config in the caller repo |
| `branch_name_pattern` | `^((feat\|fix)/.+\|revert-[0-9]+-.+)$` | Allowed PR branch names |
| `default_bump` | `patch` | Bump when no conventional-commit keyword is found |
| `tag_prefix` | `v` | Release tag prefix |
| `release_branches` | `main` | Branches that produce releases |

## Composite actions

| Action | Purpose |
|--------|---------|
| `gcr-login` | Authenticate to Google Cloud and configure Docker for GCR |
| `build-and-push` | Build a Docker image and push it to GCR |
| `check-if-image-exists-in-gcr` | Skip builds when the image tag already exists |
| `get-environment` | Map git ref to an environment name (`prod`, `staging`, `dev`, `preview-*`) |

## Versioning

Tags follow semver (`v1.0.0`); the major tag (`v1`) is moved to the latest
compatible release so callers can pin `@v1`.
