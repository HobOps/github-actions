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

### `docker-component.yml`

Build (and optionally push to GCR) a **single** Docker component. In a
monorepo, call this workflow once per component (`frontend`, `backend`, …).
Path filters / change detection stay in the caller so only changed components
run.

| Mode | When | Purpose |
|------|------|---------|
| Build-only | `push: false` (typical on PRs) | `docker build` with env + sha tags; no registry credentials needed |
| Push | `push: true` | GCR login → skip if `:github.sha` already exists → build + push |

Tags written on push (via the `build-and-push` composite action):

- `${docker_repo}/${image_name}:${github.sha}`
- `${docker_repo}/${image_name}:${environment}-${short_sha}`
- `${docker_repo}/${image_name}:${environment}-latest`

Usage (`.github/workflows/docker.yml` in a monorepo such as `docker-example-app`):

```yaml
name: Docker

on:
  pull_request:
    branches: [main]
  push:
    branches: [main, staging, develop]

concurrency:
  group: docker-${{ github.event.pull_request.number || github.ref }}
  cancel-in-progress: true

permissions:
  contents: read

jobs:
  backend:
    uses: HobOps/github-actions/.github/workflows/docker-component.yml@v1
    with:
      context: ./backend
      image_name: docker-example-app-backend
      docker_repo: ${{ vars.DOCKER_REPO }}
      # Build on PRs; push only after merge / branch pushes
      push: ${{ github.event_name == 'push' }}
    secrets:
      gcp_credentials_json: ${{ secrets.GCR_PUSH_KEY }}

  frontend:
    uses: HobOps/github-actions/.github/workflows/docker-component.yml@v1
    with:
      context: ./frontend
      image_name: docker-example-app-frontend
      docker_repo: ${{ vars.DOCKER_REPO }}
      push: ${{ github.event_name == 'push' }}
    secrets:
      gcp_credentials_json: ${{ secrets.GCR_PUSH_KEY }}
```

Optional path filters in the caller (only build what changed):

```yaml
on:
  pull_request:
    paths: [backend/**, frontend/**, .github/workflows/docker.yml]
  push:
    paths: [backend/**, frontend/**, .github/workflows/docker.yml]
```

For finer control, add a `dorny/paths-filter` job and gate each component
with `needs` + `if: needs.changes.outputs.backend == 'true'`.

Inputs:

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `context` | yes | — | Docker build context path |
| `dockerfile` | no | `<context>/Dockerfile` | Dockerfile path |
| `image_name` | yes | — | Image name without registry prefix |
| `docker_repo` | yes | — | Registry path (`gcr.io/my-project`) |
| `push` | no | `true` | Push to registry (`false` = build-only) |
| `force_build` | no | `false` | Rebuild even if `:github.sha` exists in GCR |
| `environment` | no | *(from ref)* | Tag environment; empty → `get-environment` |

Secrets:

| Secret | Required | Description |
|--------|----------|-------------|
| `gcp_credentials_json` | when `push: true` | GCP SA JSON for GCR |

Outputs: `environment`, `tag`, `tag_latest`, `image`, `built`.

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
