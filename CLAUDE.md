# shared-workflows

Reusable GitHub Actions workflows for the `weirdapps` org. The repo is public, so any weirdapps repo (public or private) can call a workflow with `uses: weirdapps/shared-workflows/.github/workflows/<name>.yml@main`.

There is no application code, no local runtime, and no self-CI in this repo. The workflows are exercised only by their callers. "Testing" a change means pointing a caller at your branch or commit SHA and watching that caller's Actions run.

## Contents

| Workflow | Purpose |
|---|---|
| `.github/workflows/deps-refresh.yml` | Node/npm dependency refresh: refresh the lockfile, run a caller-supplied gate command, open a PR when the gate passes. |
| `.github/workflows/dependabot-auto-merge.yml` | Classify a Dependabot PR, wait for the caller's checks, squash-merge safe updates. Ungrouped majors stay open; grouped majors merge unless the caller sets `allow_major_in_group: false`. |

## deps-refresh.yml contract

Inputs: `node_version` (default `lts/*`), `setup_cmd` (optional, step skipped when empty), `refresh_cmd` (default `npm update` then `npm install`), `gate_cmd` (REQUIRED, must exit 0 or no PR opens).

Secret: `PUSH_PAT` (optional, falls back to `github.token`). Use a PAT when you want a bot-authored PR, or when the PR must trigger other workflows (PRs opened by `github.token` do not trigger downstream workflows).

Output (callers depend on these exact values): branch `deps/monthly-refresh` (reused and force-updated while its PR is open, so a second run updates that PR instead of opening a new one; `delete-branch: true` removes it once no active PR references it), commit and PR title `deps: monthly dependency refresh`, label `dependencies`. Changing the branch name, title, or label is a breaking change for every caller's merge tooling; treat them as a stable contract.

## dependabot-auto-merge.yml contract

Inputs: `allow_minor` (default true), `allow_grouped` (default true), `allow_major_in_group` (default true), `exclude_branch_prefixes` (default empty, whitespace-separated), `checks_timeout_minutes` (default 45), `merge_method` (default `squash`). No secrets, uses the automatic `GITHUB_TOKEN`. Caller must grant `contents: write` and `pull-requests: write`.

Callers are NOT uniform. Before changing a default, check what each caller relied on: `outlook-access`, `plessas-marketplace`, and `teams-access` run the stricter `allow_major_in_group: false`; every other caller takes the defaults. Flattening those three is a silent policy regression, not a cleanup.

No caller currently sets `exclude_branch_prefixes`. In particular `etorotrade` is NOT a caller of this reusable: it still runs its own local `dependabot-auto-merge.yml` built on `gh pr merge --auto --squash` with an inline `dependabot/pip/` guard, so changes here do not reach it. Do not cite it as a consumer.

Three invariants that are easy to break and expensive to debug:

1. **Do not switch this back to a bare `gh pr merge --auto`.** Native auto-merge needs `allow_auto_merge` AND a branch-protection rule. On this plan, private repos cannot have branch protection (`403 Upgrade to GitHub Pro`) and `PATCH allow_auto_merge=true` silently no-ops on them, returning `200` with the value still `false`. `--auto` is attempted first as a fast path, but the polling fallback is what actually merges on most repos here.
2. **The check poll must keep excluding `.workflow != github.workflow`.** This job is itself a check on the PR it is merging. Waiting on "all checks" waits on itself and hangs until the job timeout.
3. **The grouped block is the sole decider for grouped PRs and can both grant and revoke.** It runs after the `case` on update type and overwrites that verdict, so `allow_minor: false` does NOT yield patch-only behaviour on a grouped PR. Verified by running the classify script: update-type `semver-minor` with a non-empty group and `allow_minor: false` still classifies `merge=true`.

Majors are handled two different ways, and that is deliberate. An ungrouped major is never auto-merged. A grouped update whose aggregate level is major IS auto-merged under the defaults, because `allow_major_in_group` defaults to true; only the three repos listed above opt out. Do not "fix" the classify block to hold every major: that silently changes behaviour for every caller on the defaults.

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
