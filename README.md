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

## MCP Registry publish and private images

The `mcp-registry` job publishes `server.json` to registry.modelcontextprotocol.io.
The registry accepts only images anyone can pull without logging in, and GHCR
creates every new container package as **private**. Until the package is made
public, a server's first release fails at the registry step.

The job now checks for this before publishing. If the image can't be pulled
anonymously, the job fails with a `GHCR package is private` error and a link to
the package settings page. To fix it, a package admin opens
`https://github.com/orgs/WYRE-AI/packages/container/package/<server>/settings`,
goes to **Danger Zone → Change visibility** and sets it to **Public**. GitHub has
no API for this. Then re-run the failed job.

If a server should stay private (for example its repo is private), the caller
turns the registry step off:

```yaml
    uses: WYRE-AI/reusable-workflows/.github/workflows/mcp-server-release.yml@<sha>
    with:
      server-name: example-mcp
      image-name: ghcr.io/wyre-ai/example-mcp
      publish-to-registry: false
```

Docker push, the MCPB bundle and the security scan run either way.
