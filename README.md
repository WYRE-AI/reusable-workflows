# WYRE-AI/reusable-workflows

Public host for the reusable GitHub Actions workflows called by the WYRE-AI
`*-mcp` server fleet. These live here — not in `WYRE-AI/.github` — because
GitHub requires a reusable workflow's host repository to be public when the
calling repository is public, and `WYRE-AI/.github` is private.

Only reusable workflows belong in this repo. Everything else (org community
health files, the activity notifier, docs) stays in `WYRE-AI/.github`.

## Workflows

- `.github/workflows/mcp-server-release.yml` — semantic-release → Docker/GHCR →
  MCPB bundle → MCP Registry publish → Trivy scan.
- `.github/workflows/mcp-server-deploy.yml` — Azure Container Apps deploy via
  OIDC, by immutable image digest.
- `.github/workflows/mcp-server-verify.yml` — build, lint, test only, with
  read-only permissions. For callers' `pull_request` runs (see below).

Callers pin by SHA:

```yaml
uses: WYRE-AI/reusable-workflows/.github/workflows/mcp-server-release.yml@<sha>
```

## Least privilege on pull_request

`mcp-server-release.yml` runs `verify` (build, lint, test) on every event and
everything else on push to main only. Inside this file the `verify` job already
declares `contents: read` + `packages: read`, uses no secret except the
automatic `GITHUB_TOKEN`, checks out with `persist-credentials: false` and
installs with `npm ci --ignore-scripts`. Publishing jobs are gated on
`github.event_name == 'push'`.

A reusable workflow cannot widen or remove the **caller's** grant, though. If
the calling job grants `contents: write`, `id-token: write` and friends plus
`secrets: inherit` and also fires on `pull_request`, those are the ceiling
for the PR run. And a caller job with read-only permissions cannot call
`mcp-server-release.yml` at all: its nested jobs request write scopes, and
GitHub rejects that at validation time even when the jobs would be skipped.

To drop the grant on pull requests, split the caller into one job per event.
The PR job calls `mcp-server-verify.yml`; the push job calls the release
workflow:

```yaml
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  verify:
    if: github.event_name == 'pull_request'
    permissions:
      contents: read
      packages: read
    uses: WYRE-AI/reusable-workflows/.github/workflows/mcp-server-verify.yml@<sha>

  release:
    if: github.event_name == 'push'
    permissions:
      actions: read
      contents: write
      issues: write
      pull-requests: write
      packages: write
      id-token: write
      security-events: write
    uses: WYRE-AI/reusable-workflows/.github/workflows/mcp-server-release.yml@<sha>
    with:
      server-name: example-mcp
      image-name: ghcr.io/wyre-ai/example-mcp
    # no `secrets: inherit`: the workflow only uses GITHUB_TOKEN
```

Notes for the caller change:

- The PR check name changes from `release / Build, Lint, Test` to
  `verify / Build, Lint, Test`. Update any branch protection or ruleset that
  requires the old name, or PRs will wait for a check that never reports.
  **Order matters:** the repin PR itself reports only the new name, so it cannot
  merge while branch protection still requires the old one. Drop the old
  required check from branch protection before merging the caller change, then
  add `verify / Build, Lint, Test`. As of 2026-11-11 four repos require the old
  name: `syncro-mcp`, `ninjaone-mcp`, `nutanix-mcp` and `axcient-mcp`.
- Any `deploy` job that has `needs: release` keeps working: it only matters on
  push, where `release` still runs.
- `secrets: inherit` can be dropped from the push job too, for the release
  workflow only. It reads nothing but `GITHUB_TOKEN`, which is always
  available. `mcp-server-deploy.yml` is different: it declares its own
  `secrets:` and needs `AZURE_CLIENT_ID`, `AZURE_TENANT_ID` and
  `AZURE_SUBSCRIPTION_ID`, so a `deploy` job keeps them.
- A caller-level `permissions:` block cannot be made conditional on the event:
  `permissions` does not accept expressions. Separate jobs are the way.

Provenance: copied verbatim from `WYRE-AI/.github` at
`8c2360058e882a8afacfc4cebf840135789ef406` (itself adapted 2026-08-24 from the
public `wyre-technology/.github` during the org migration).
