# shared-workflows

Reusable GitHub Actions workflows shared across the `weirdapps` repos.
Hosted in a normal public repo so any repo (public or private) can call them.

## `deps-refresh.yml`

Monthly dependency-refresh engine: refresh a repo's lockfile → validate
(tests/build) → open a PR for review. One source of truth instead of a copy
per repo.

**Caller** (`<repo>/.github/workflows/deps-refresh.yml`):

```yaml
name: Monthly Dependency Refresh
on:
  schedule:
    - cron: "37 6 18 * *"   # staggered per repo
  workflow_dispatch: {}
jobs:
  refresh:
    uses: weirdapps/shared-workflows/.github/workflows/deps-refresh.yml@main
    with:
      runtime: node
      node_version: "20"
      refresh_cmd: |
        npm update
        npm install
      gate_cmd: npm test
    secrets: inherit
```

### Inputs
| input | required | description |
|-------|----------|-------------|
| `runtime` | yes | `node` or `python` |
| `node_version` | no | Node version (default `lts/*`) |
| `python_version` | no | Python version (default `3.12`) |
| `setup_cmd` | no | extra setup before refresh (e.g. install poetry/uv) |
| `refresh_cmd` | yes | command that refreshes the lockfile |
| `gate_cmd` | yes | validation command that must pass before the PR opens |

Optional `PUSH_PAT` secret (falls back to `GITHUB_TOKEN`). The caller owns the
schedule; this workflow owns the logic. Command inputs run via env vars
(`bash -c "$VAR"`), never interpolated into a `run:` block.

> The npm repos call this. The Python-lockfile repos (etorotrade/poetry,
> etoro_statement/pip-tools, loans/uv) keep bespoke standalone workflows —
> each has a unique relock mechanism, so they gain little from sharing.
