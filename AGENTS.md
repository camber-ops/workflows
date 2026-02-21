# AGENTS.md — camber-ops/workflows

## Purpose

This repository contains **reusable GitHub Actions workflows** for the `camber-ops`
organization. All workflows use the `workflow_call` trigger and are intended to be
called from other repositories via:

```yaml
uses: camber-ops/workflows/.github/workflows/<workflow-file>.yaml@main
```

---

## Repository Structure

```
.github/workflows/   # All reusable workflow definitions
.yamllint.yaml       # yamllint configuration used across the org
```

---

## Workflow Catalog

### CI Workflows

| File | Purpose |
|------|---------|
| `ci.python.yaml` | Python quality checks (flake8, pydocstyle, radon, mypy) + tests. Uses **Poetry**. Older standard. |
| `ci.python-check.yaml` | Python quality checks (ruff, mypy) + tests + coverage comments. Uses **uv**. Newer standard. |
| `ci.python-gemini.yaml` | Python checks inside the `gemini` ECR container. Modifies `pyproject.toml` for `##CI-ONLY##` deps before running. Uses **Poetry**. |
| `ci.python-horizon.yaml` | Python checks inside the `horizon` ECR container. Same pattern as gemini. Uses **Poetry**. |
| `ci.structure.yaml` | Enforces **single commit per PR** using the GitHub REST API. |
| `ci.yaml.yaml` | Runs `yamllint` against all YAML files. Falls back to fetching `.yamllint.yaml` from this repo if none is present in the caller. |

### CD Workflows

| File | Purpose |
|------|---------|
| `cd.pipper.yaml` | Publishes Python packages to the internal **Pipper** registry. Uses **Poetry**. |
| `cd.pipper-uv.yaml` | Publishes Python packages to the internal **Pipper** registry. Uses **uv**. Newer standard. |
| `cd.code-artifact.yaml` | Publishes Python packages to **AWS CodeArtifact**. Uses **uv**. Supersedes Pipper for new projects. |
| `cd.docker.yaml` | Builds and pushes Docker images to ECR (public + private). Uses **Podman** as the Docker runtime. Supports `amd64` (default) and `arm64` targets. |
| `cd.artifacts.yaml` | Downloads a GitHub Actions artifact and uploads it to S3 (`cbrdata-devel-artifacts` or `cbrdata-prod-artifacts`). |
| `cd.docsify.yaml` | Runs `docsifier collect` to bundle docs and uploads them to S3 via `cd.artifacts.yaml`. |
| `cd.lambda.yaml` | Runs a `reviser` shell macro inside the `proxy-hub-prod:reviser-*` ECR container to deploy Lambda functions. |
| `cd.terraform.yaml` | Runs `terraform fmt --check`, `ci tf plan`, and optionally `ci tf apply` for `devel` and/or `prod` workspaces. Uses the `gemini:ci-terraform-prod` ECR container. |

### Internal Workflow

| File | Purpose |
|------|---------|
| `yaml-all.yaml` | Self-check — runs `ci.yaml.yaml` on any YAML change pushed or PR'd to `main` in this repo. |

---

## Key Conventions

### Package Managers
- **uv** (`ci.python-check.yaml`, `cd.pipper-uv.yaml`) — modern standard for new
  projects.
- **Poetry** (`ci.python.yaml`, `cd.pipper.yaml`, container-based workflows) — legacy
  standard, still in active use.
- When adding a new Python workflow, prefer **uv** unless the caller indicates that
  Poetry should be used.

### Internal Package Registry — Pipper
- Camber uses **Pipper**, an S3-backed internal Python package registry.
- Bucket: `cbrdata-pipper` (internal packages),
  `cbrdata-devel-artifacts` / `cbrdata-prod-artifacts` (build artifacts).
- Callers typically expose `pipper_update`, `pipper_update_all`, `pipper_bundle`,
  `pipper_publish`, `pipper_publish_devel` as Taskipy tasks.
- **Pipper is the legacy standard.** New projects should use CodeArtifact instead.

### Package Registry — AWS CodeArtifact
- CodeArtifact is the modern standard for internal Python package distribution.
- Domain: `camber`. Devel repo: `camber-python-devel`. Prod repo: `camber-python`.
- Publishing is handled natively in the workflow via `aws codeartifact` token
  retrieval and `uv publish` — no Taskipy tasks are required in the caller.
- Requires the `gh-code-artifact` IAM role to be provisioned in the AWS account.

### AWS Authentication
- All AWS access uses **OIDC** (no long-lived credentials).
- Every workflow that needs AWS includes:
  ```yaml
  permissions:
    id-token: write
    contents: read
  ```
- The only secrets expected from callers are `AWS_ACCOUNT_ID` (for role ARNs) and
  `ECR_TOKEN` (for pulling private ECR container images).

### IAM Roles (all in `arn:aws:iam::<AWS_ACCOUNT_ID>:role/`)
| Role | Used by |
|------|---------|
| `gh-standard` | General-purpose read (Pipper installs in CI) |
| `gh-pipper` | Publishing to the Pipper registry |
| `gh-code-artifact` | Publishing to AWS CodeArtifact (`cd.code-artifact.yaml`) |
| `gh-ecr` | Pushing images to ECR |
| `gh-artifacts` | Uploading to artifact S3 buckets |
| `gh-lambda` | Deploying Lambda functions |
| `gh-aws-read` | Terraform read-only plan |
| `gh-aws-read-write` | Terraform apply |

### Docker / Container Builds
- Uses **Podman** aliased to `docker` (not Docker daemon).
- Linting done with **Hadolint** v2.14.0 via `ci hadolint <target>`.
- Build/push done via `ci build <mode> <target> --push --podman`.
- `build_mode` is one of `ci`, `devel`, or `prod`.
- ARM builds require a custom runner (`ubuntu-arm-custom`); AMD builds use
  `ubuntu-latest`.
- `build_target` is a space-separated list of sequential targets.

### Python Quality Tools
| Toolchain | Linters | Type Checker | Test Runner |
|-----------|---------|--------------|-------------|
| uv (modern) | ruff | mypy | pytest via `task test` |
| Poetry (legacy) | flake8, pydocstyle, radon | mypy | pytest via `task test` |

### Terraform
- Terraform state is managed per-workspace (`devel`/`prod`).
- `ci tf <env> plan/apply` is a wrapper CLI available inside the
  `gemini:ci-terraform-prod` container.
- `devel`/`prod` inputs accept `"skip"`, `"plan"`, or `"apply"`.

### PR Policy
- `ci.structure.yaml` enforces **exactly one commit per PR**. Squash/rebase before
  merging.

### YAML Style
- `.yamllint.yaml` at the repo root defines org-wide YAML linting rules.
- Callers that don't have their own `.yamllint.yaml` will inherit this repo's config
  automatically via `ci.yaml.yaml`.
- Use `uvx yamllint` command when validation yaml files in this repo.

### Markdown Style
- Hard line limit of 89 characters where possible for all files in this repo.

---

## Secrets Reference

| Secret | Required By | Description |
|--------|-------------|-------------|
| `AWS_ACCOUNT_ID` | All AWS workflows | AWS account ID for role ARN construction |
| `ECR_TOKEN` | Container-based workflows | Password to pull private ECR images |
| `GITHUB_TOKEN` | `ci.structure.yaml`, `ci.python-check.yaml` | Implicit GitHub token for API calls and PR comments |

---

## Adding a New Workflow

1. Create `.github/workflows/ci.<name>.yaml` or `cd.<name>.yaml`.
2. Use `on: workflow_call` as the trigger.
3. Include the OIDC permissions block if AWS access is needed.
4. Reference `AWS_ACCOUNT_ID` from secrets (never hardcode account IDs).
5. Prefer **uv** over Poetry for new Python workflows.
6. Validate locally with `yamllint .` before pushing.

---

## ECR Image Registry

Private ECR account: `819888084929.dkr.ecr.us-west-2.amazonaws.com`

| Image | Used In |
|-------|---------|
| `gemini:gemini-workflows-prod-prerelease` | `ci.python-gemini.yaml` |
| `gemini:ci-terraform-prod` | `cd.terraform.yaml` |
| `horizon:horizon-prod-prerelease` | `ci.python-horizon.yaml` |
| `proxy-hub-prod:reviser-<version>` | `cd.lambda.yaml` |
