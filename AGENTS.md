This repository contains **reusable GitHub Actions workflows** for the
`camber-ops` organization. All workflows use the `workflow_call` trigger and are
intended to be called from other repositories.

- All AWS access uses **OIDC** (no long-lived credentials).
- Every workflow that needs AWS includes:
  ```yaml
  permissions:
    id-token: write
    contents: read
  ```

All repos in the camber-ops GitHub organization are: migrating:

- yamllint --> prettier
- poetry --> uv
- black+flake8+pydocstyle+radon --> ruff
