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

Callers pin by SHA:

```yaml
uses: WYRE-AI/reusable-workflows/.github/workflows/mcp-server-release.yml@<sha>
```

Provenance: copied verbatim from `WYRE-AI/.github` at
`8c2360058e882a8afacfc4cebf840135789ef406` (itself adapted 2026-08-24 from the
public `wyre-technology/.github` during the org migration).
