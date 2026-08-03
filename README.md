# shared-workflows

Reusable GitHub Actions workflows for the `weirdapps` org.

The repo is public so any weirdapps repo (public or private) can call the workflows via `uses:`.

There is no CI in this repo itself; the workflows are exercised by their callers.

## Contents

| Workflow | Purpose |
|---|---|
| [`.github/workflows/deps-refresh.yml`](.github/workflows/deps-refresh.yml) | Node/npm dependency refresh: refresh the lockfile, run a validation command, open a PR with the changes. |
| [`.github/workflows/dependabot-auto-merge.yml`](.github/workflows/dependabot-auto-merge.yml) | Classify a Dependabot PR, wait for the caller's checks, squash-merge safe updates. Works without branch protection. |

## `deps-refresh.yml`

A Node-focused reusable workflow that:

1. Checks out the caller repo (`actions/checkout` v6, pinned to a commit SHA).
2. Sets up Node (`actions/setup-node` v6, pinned to a commit SHA) at the requested version.
3. Optionally runs a caller-supplied `setup_cmd` (extra tooling install, etc.). Skipped when the input is empty.
4. Runs `refresh_cmd` to refresh the lockfile. Defaults to `npm update` followed by `npm install`.
5. Runs `gate_cmd` to validate the refreshed tree. If it exits non-zero, the workflow fails and no PR opens.
6. Opens a PR via `peter-evans/create-pull-request` v8.1.1 (pinned to a commit SHA) with a fixed branch, title, and label (see [Output](#output)).

Command inputs execute through env vars using `bash -euo pipefail -c "$VAR"`, never interpolated directly into a `run:` block, so caller-supplied strings cannot inject workflow syntax.

### Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `node_version` | no | `lts/*` | Node.js version passed to `actions/setup-node`. |
| `setup_cmd` | no | (empty) | Extra shell command to run before the refresh. Step is skipped when empty. |
| `refresh_cmd` | no | `npm update` then `npm install` | Command that refreshes the lockfile. |
| `gate_cmd` | yes | (none) | Validation command. Must exit 0 for the PR to open. |

### Secrets

| Secret | Required | Description |
|---|---|---|
| `PUSH_PAT` | no | PAT used by the PR step. Falls back to `github.token` when not provided. Use a PAT if you want the PR authored by a bot account rather than `github-actions[bot]`, or if you need cross-workflow triggers (PRs opened by `github.token` do not trigger other workflows). |

### Required caller permissions

The reusable workflow declares:

```yaml
permissions:
  contents: write
  pull-requests: write
```

A reusable workflow cannot escalate beyond the permissions the caller grants, so the caller job MUST also declare both. If either is missing, the checkout or PR step will fail.

### Caller example

Drop this into a caller repo at `.github/workflows/deps-refresh.yml`:

```yaml
name: Monthly Dependency Refresh

on:
  schedule:
    - cron: "37 6 18 * *"   # 06:37 UTC on the 18th of each month
  workflow_dispatch: {}

jobs:
  refresh:
    permissions:
      contents: write
      pull-requests: write
    uses: weirdapps/shared-workflows/.github/workflows/deps-refresh.yml@main
    with:
      node_version: "20"
      gate_cmd: npm test
    secrets: inherit
```

A caller that needs extra tooling before the refresh, or a non-default refresh command, can pass `setup_cmd` and override `refresh_cmd`:

```yaml
    with:
      node_version: "20"
      setup_cmd: npm ci
      refresh_cmd: |
        npm update
        npm install
      gate_cmd: npm run test:ci
```

Stagger the cron across repos so the shared runner minutes are not all consumed on the same day.

### Output

When `gate_cmd` passes, one PR is opened per run:

- Branch: `deps/monthly-refresh` (recreated each run via `delete-branch: true`).
- Commit and PR title: `deps: monthly dependency refresh`.
- Label: `dependencies`.
- Body: a short note that the refresh was produced by this shared reusable and that validation passed.

When `gate_cmd` fails, the job fails and no PR is opened.

## `dependabot-auto-merge.yml`

Classifies a Dependabot PR, waits for the caller's own checks, and squash-merges the safe ones. Major bumps are left open for manual review.

### Why this exists

GitHub's native auto-merge (`gh pr merge --auto`) requires **both** `allow_auto_merge` on the repository **and** a branch-protection rule to gate on. Neither is reliably available in this org:

- Branch protection on a **private** repo is a paid feature on the current plan, so `GET /repos/{r}/branches/{b}/protection` returns `403 Upgrade to GitHub Pro`.
- `PATCH /repos/{r} allow_auto_merge=true` **silently no-ops** on a private repo. It returns `200` with `allow_auto_merge: false`.

So `--auto` fails with either `Auto merge is not allowed for this repository` or `Protected branch rules not configured for this branch`, the job exits non-zero, and every patch and minor Dependabot PR piles up indefinitely. This workflow tries `--auto` first (cheapest and correct where it does work) and otherwise polls the PR's checks itself and merges once they are green.

The poll **excludes the caller workflow's own checks**, matched on `.workflow != github.workflow`. This job is itself a check on the same PR, so waiting for "all checks" would wait for itself and deadlock until the job timeout.

### Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `allow_minor` | no | `true` | Auto-merge `semver-minor` and `semver-patchminor`. Set `false` for patch-only. |
| `allow_grouped` | no | `true` | Auto-merge grouped updates. Set `false` to leave every group for manual review. |
| `allow_major_in_group` | no | `true` | Auto-merge a group whose aggregate level is `semver-major`. Set `false` for the stricter "a major is never auto-merged, not even inside a group" policy while still auto-merging safe groups. Only consulted when `allow_grouped` is true. |
| `exclude_branch_prefixes` | no | `""` | Whitespace-separated head-branch prefixes to skip, for example `dependabot/pip/`. Use where Dependabot edits a *derived* lockfile and would desync the real source of truth. |
| `checks_timeout_minutes` | no | `45` | How long to wait for the caller's checks before failing. |
| `merge_method` | no | `squash` | `squash`, `merge`, or `rebase`. |

### Secrets

None. Uses the automatic `GITHUB_TOKEN`.

### Required caller permissions

The caller job must grant at least what the reusable declares, because a reusable workflow cannot escalate:

```yaml
permissions:
  contents: write
  pull-requests: write
```

### Caller example

```yaml
name: Dependabot auto-merge
'on': pull_request

permissions:
  contents: write
  pull-requests: write

jobs:
  auto-merge:
    uses: weirdapps/shared-workflows/.github/workflows/dependabot-auto-merge.yml@main
```

With a repo-specific exclusion (this is the `etorotrade` case, where `poetry.lock` is the source of truth and pip-Dependabot edits only the derived `requirements-*-lock.txt`):

```yaml
jobs:
  auto-merge:
    uses: weirdapps/shared-workflows/.github/workflows/dependabot-auto-merge.yml@main
    with:
      exclude_branch_prefixes: "dependabot/pip/"
```

With the stricter major policy (`outlook-access`, `plessas-marketplace`, `teams-access`), where a major must never land unreviewed even as part of a group:

```yaml
jobs:
  auto-merge:
    uses: weirdapps/shared-workflows/.github/workflows/dependabot-auto-merge.yml@main
    with:
      allow_major_in_group: false
```

### Behaviour

- Not a Dependabot PR: the job is skipped by its `if:` guard.
- Major bump, or an excluded branch prefix: classified `merge=false`, a note is written to the step summary, PR stays open.
- Any non-self check fails: the failing `workflow / job` names are printed and the job exits non-zero. Nothing is merged.
- Checks still pending past `checks_timeout_minutes`: the job fails rather than merging blind.
- Caller has no CI at all: only the self-check exists, so the poll sees zero other checks and merges immediately.
- Merge loses a race to another PR: retried three times, then the workflow comments `@dependabot rebase` and exits cleanly. Dependabot's rebase re-triggers the workflow.

## Flow

```mermaid
flowchart TD
  A[Caller repo: schedule or workflow_dispatch] --> B[shared-workflows/deps-refresh.yml]
  B --> C[actions/checkout v6]
  C --> D[actions/setup-node v6]
  D --> E{setup_cmd set?}
  E -- yes --> F[bash -c setup_cmd]
  E -- no --> G[bash -c refresh_cmd]
  F --> G
  G --> H[bash -c gate_cmd]
  H -- pass --> I[peter-evans/create-pull-request v8.1.1]
  H -- fail --> X[Job fails, no PR]
  I --> J[PR on deps/monthly-refresh]
```

## Versioning

There are no tagged releases. Callers pin with either:

- `@main` to track the tip. Changes propagate immediately, which is fine for a small consumer set.
- `@<commit-sha>` for immutable pinning.

When making a breaking change here, switch pinned callers to a SHA on the previous commit first, then merge the change.

## Adding a new reusable workflow

1. Create `.github/workflows/<name>.yml` with an `on: workflow_call:` trigger.
2. Read any caller-supplied commands through `env:` and run them via `bash -euo pipefail -c "$VAR"`. Do not interpolate `${{ inputs.foo }}` directly into a `run:` block.
3. Pin third-party actions to a commit SHA and put the readable version in a comment above the `uses:` line.
4. Declare only the permissions the workflow actually needs, and document that the caller must grant them too (a reusable cannot escalate).
5. Add a section to this README documenting purpose, inputs, secrets, required caller permissions, and a caller example.

## Repo layout

```
.
├── .github/
│   └── workflows/
│       └── deps-refresh.yml
└── README.md
```

## License

MIT, see [LICENSE](LICENSE). Copyright (c) 2026 Dimitrios Plessas.
