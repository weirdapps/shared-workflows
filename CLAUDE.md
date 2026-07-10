# shared-workflows

Reusable GitHub Actions workflows for the `weirdapps` org. The repo is public, so any weirdapps repo (public or private) can call a workflow with `uses: weirdapps/shared-workflows/.github/workflows/<name>.yml@main`.

There is no application code, no local runtime, and no self-CI in this repo. The workflows are exercised only by their callers. "Testing" a change means pointing a caller at your branch or commit SHA and watching that caller's Actions run.

## Contents

| Workflow | Purpose |
|---|---|
| `.github/workflows/deps-refresh.yml` | Node/npm dependency refresh: refresh the lockfile, run a caller-supplied gate command, open a PR when the gate passes. |

## deps-refresh.yml contract

Inputs: `node_version` (default `lts/*`), `setup_cmd` (optional, step skipped when empty), `refresh_cmd` (default `npm update` then `npm install`), `gate_cmd` (REQUIRED, must exit 0 or no PR opens).

Secret: `PUSH_PAT` (optional, falls back to `github.token`). Use a PAT when you want a bot-authored PR, or when the PR must trigger other workflows (PRs opened by `github.token` do not trigger downstream workflows).

Output (callers depend on these exact values): branch `deps/monthly-refresh` (recreated each run), commit and PR title `deps: monthly dependency refresh`, label `dependencies`. Changing the branch name, title, or label is a breaking change for every caller's merge tooling; treat them as a stable contract.

## Key conventions (MUST follow when editing)

- Never interpolate `${{ inputs.* }}` into a `run:` block. Read caller-supplied commands through `env:` and execute via `bash -euo pipefail -c "$VAR"`. Direct interpolation lets a caller inject workflow syntax; this env-indirection is the repo's core security property.
- Pin every third-party action to a full commit SHA, with the human-readable version in a comment above the `uses:` line (for example `# v6` then `uses: actions/checkout@<sha>`).
- Declare only the permissions a workflow needs (`contents: write`, `pull-requests: write` for deps-refresh) and document that the caller job must grant the same. A reusable workflow cannot escalate beyond what the caller grants.
- No tagged releases. Callers pin `@main` (tracks tip, propagates immediately) or `@<sha>` (immutable). For a breaking change, move pinned callers to a SHA on the previous commit first, then merge.

## Adding a new reusable workflow

1. Create `.github/workflows/<name>.yml` with an `on: workflow_call:` trigger.
2. Read caller commands through `env:`; run them via `bash -euo pipefail -c "$VAR"`.
3. Pin third-party actions to a commit SHA (readable version in a comment).
4. Declare minimal permissions; document that the caller must grant them.
5. Document purpose, inputs, secrets, required caller permissions, and a caller example in `README.md`.

## Consumers

Callers drop a thin wrapper at their own `.github/workflows/deps-refresh.yml` that calls this workflow on a monthly cron (staggered across repos to spread runner minutes) plus `workflow_dispatch`. Full caller examples are in `README.md`.

## License

MIT. Copyright (c) 2026 Dimitrios Plessas.
