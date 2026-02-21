# CD Code Artifact

## Overview

Publishes Python packages to **AWS CodeArtifact** for use in other repositories.
CodeArtifact is the org's new standard for internal Python package distribution,
superseding the Pipper S3-backed registry. This workflow is the CodeArtifact
equivalent of `cd.pipper-uv.yaml` and uses the same `uv`-based toolchain.

---

## Requirements

1. The workflow must accept `devel` and `prod` boolean inputs (both default `false`)
   that control which CodeArtifact repository the package is published to.
2. The workflow must accept a `python_version` string input (default `"3.14"`) to
   control the Python interpreter used for the build.
3. The workflow must accept an `install_pipper` boolean input (default `true`). When
   `true`, the workflow runs `uv run task pipper_update_all` before building, so
   callers that still depend on Pipper packages during the build can install them.
4. The workflow must authenticate to AWS using OIDC, assuming the `gh-code-artifact`
   IAM role constructed from the `AWS_ACCOUNT_ID` secret.
5. The workflow must build the Python distribution with `uv build`.
6. When `devel` is `true`, the workflow must publish to the `camber-python-devel`
   CodeArtifact repository in the `camber` domain.
7. When `prod` is `true`, the workflow must publish to the `camber-python`
   CodeArtifact repository in the `camber` domain.
8. Publishing must use `uv publish` with a short-lived CodeArtifact authorization
   token obtained via `aws codeartifact get-authorization-token`. The repository
   endpoint must be resolved via `aws codeartifact get-repository-endpoint`.

---

## Acceptance Criteria

- Given a calling repo with a valid `pyproject.toml`, when the workflow runs with
  `devel: true`, the package version is visible in the `camber-python-devel`
  CodeArtifact repository.
- Given a calling repo with a valid `pyproject.toml`, when the workflow runs with
  `prod: true`, the package version is visible in the `camber-python` CodeArtifact
  repository.
- Given `install_pipper: false`, no `uv run task pipper_update_all` step executes.
- Given `install_pipper: true`, the Pipper dependencies step runs before `uv build`.
- When both `devel` and `prod` are `false`, no package is published (build-only run).

---

## Out of Scope

- Migration tooling from Pipper to CodeArtifact.
- Multi-domain CodeArtifact support (only the `camber` domain is in scope).
- Non-Python artifact types.
- Creating or managing the `gh-code-artifact` IAM role — this is an AWS
  infrastructure prerequisite that must be provisioned before the workflow is usable.

---

## Testing Strategy

GitHub Actions reusable workflows cannot be unit tested in isolation. Validation for
this workflow uses two levels:

1. **YAML lint**: `yamllint .` must pass locally before any push. The `ci.yaml.yaml`
   workflow enforces this automatically on PR.
2. **Integration smoke test**: A calling repo invokes the workflow with `devel: true`
   against a test branch. Successful completion is verified by confirming the package
   version appears in `camber-python-devel` in CodeArtifact.

---

## Implementation Notes

### Implementation Decisions

- **Direct publish, no Taskipy tasks**: Unlike `cd.pipper-uv.yaml`, which delegates
  to caller-defined Taskipy tasks (`pipper_bundle`, `pipper_publish`, etc.), this
  workflow handles all publish steps natively. CodeArtifact requires a short-lived
  auth token and a computed endpoint URL — these are infrastructure concerns that
  belong in the workflow, not in every caller's `Taskfile`. This also removes the
  requirement for callers to define any CA-specific tasks.
- **Token + endpoint per publish step**: The auth token and repository endpoint are
  resolved inline within each conditional publish step rather than in shared env
  variables. This keeps each step self-contained and avoids exposing the token in the
  job environment for longer than necessary.
- **`uv publish --publish-url "${ENDPOINT}"`**: The endpoint returned by
  `aws codeartifact get-repository-endpoint` is the full upload URL for
  `uv publish`. Do **not** append `/legacy/` — that suffix is PyPI.org-specific
  and returns 404 on CodeArtifact.
- **`install_pipper` retained**: Some repos still depend on Pipper packages during
  the build phase (e.g., internal libraries not yet migrated to CodeArtifact). The
  `install_pipper` input was kept with a default of `true` to mirror
  `cd.pipper-uv.yaml` and avoid breaking callers that migrate incrementally.
- **`shared` input dropped**: Pipper's `shared` publish target has no CodeArtifact
  equivalent at this time. It was intentionally omitted.
- **Delete-before-publish for devel**: A "Delete existing version from CodeArtifact
  (devel)" step runs before each devel publish, removing any previously uploaded
  version from `camber-python-devel`. This allows repeated PR workflow runs against
  the same package version to succeed without manual cleanup. The delete uses
  `|| true` so that first-time publishes (where the version does not yet exist)
  remain a no-op rather than a failure.
- **Pre-flight version check for prod**: A "Check for existing version in
  CodeArtifact (prod)" step runs before the prod publish step. If the version
  already exists in `camber-python`, the step exits with a human-readable message
  instructing the developer to bump the version in `pyproject.toml` before merging.
  This surfaces the conflict before any upload traffic and prevents silent
  overwrites in production.
- **Package name normalisation**: Both steps derive the package name from
  `pyproject.toml` via `tomllib` (stdlib, Python 3.11+) and replace underscores
  with hyphens to match PyPI/CodeArtifact normalisation. The version is read via
  `uv version --short`.

### Patterns and Conventions Introduced

- This workflow establishes the org pattern for CodeArtifact publishing: OIDC via
  `gh-code-artifact`, `uv build`, then token/endpoint resolution per target repo.
- The `camber` domain and repo names (`camber-python-devel`, `camber-python`) are
  hardcoded as org-wide constants. Do not parameterise these unless the org adopts
  multiple domains or a per-team repo structure.
- **CodeArtifact token before `uv sync`**: AWS credentials must be configured
  and a CodeArtifact token written to `UV_INDEX_CODE_ARTIFACT_PASSWORD` (via
  `$GITHUB_ENV`) *before* `uv sync` runs. Any workflow that calls `uv sync` in
  a repo with a `[[tool.uv.index]] name = "code-artifact"` entry must follow
  this ordering. All `gh-*` roles carry the required read permissions.

### Testing Approach

- YAML lint validated with `uvx yamllint .github/workflows/cd.code-artifact.yaml` —
  zero violations.
- End-to-end validation requires an integration smoke test from a calling repository
  (see Testing Strategy above).

### Future Interpretation Guidance

- The `gh-code-artifact` IAM role is a hard prerequisite. The workflow will fail at
  the `configure-aws-credentials` step until this role exists in the target AWS
  account with CodeArtifact publish permissions attached.
- If the org migrates pipper dependencies to CodeArtifact, the `install_pipper` input
  and its associated step can be removed once all callers have migrated.
