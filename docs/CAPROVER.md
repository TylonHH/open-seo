# CapRover (single-container compatibility edition)

This fork reuses upstream's `deploy/docker/Dockerfile` without application changes.
It is the legacy Wrangler-based edition, not the newer celld standalone stack.
The standalone stack needs S3, initialization, a crawl gateway and nginx.

## Images

The publish-container workflow publishes `ghcr.io/tylonhh/open-seo` on pushes to
`main`, `v*` tags, and manual dispatch. Tags: `latest` on the default branch,
`sha-<full commit SHA>`, and the exact `v*` tag. It verifies the pushed manifest
and records its digest in the run summary. Builds never deploy or synchronize
upstream automatically. Only `linux/amd64` is built; confirm NAS architecture.

GHCR packages can initially be private. Make the package public for anonymous
pulls, or configure CapRover registry credentials with `read:packages` access.
Builds use `GITHUB_TOKEN`; no personal token needed. Prefer immutable digest
references such as `ghcr.io/tylonhh/open-seo@sha256:...` for deployment.

## CapRover settings

- Container HTTP port `3001`; one instance, persistent data enabled.
- Persistent directory `/app/.wrangler`: database and local worker state.
  Back up before upgrades; migrations run automatically on startup.
- Required: `PORT=3001`, `AUTH_MODE=password`,
  `BETTER_AUTH_URL=https://<your-domain>`, stable random `BETTER_AUTH_SECRET`
  with at least 32 characters.
- Recommended: `VITE_SHOW_DEVTOOLS=false`, `OPENSEO_TELEMETRY_DISABLED=1`.
- Enable HTTPS. Restrict access until the first administrator is created:
  the first visitor claims it; later accounts require invitations.

Use CapRover's image deployment field only after target approval.
`captain-definition` also supports source builds using the upstream Dockerfile.
Do not publish host port 3001 or mount the Docker socket.

This image installs dependencies at build time but compiles Vite at first boot
and after build-relevant environment changes. Allow several minutes and adequate
RAM; upstream health checks allow 300 seconds. This is not a fully precompiled
production runtime. Upstream runs as root inside the container. `/api/health`
is public. Never expose `AUTH_MODE=local_noauth` without separate authentication
or a private network.

## External services

`DATAFORSEO_API_KEY`: base64 of `email:password` for paid DataForSEO features.
`OPENROUTER_API_KEY`: paid AI features; optional `OPENROUTER_BASE_URL` and
`OPENROUTER_MODEL`. Google Search Console and Analytics need `GOOGLE_CLIENT_ID`,
`GOOGLE_CLIENT_SECRET`, consent configuration and the stable auth secret; see
upstream Google self-hosting guides for redirect URIs. Optional `CONTEXT_API_KEY`
enables JavaScript-rendered site audits. No Cloudflare account, external database
or S3 service is required for this compatibility edition. Missing provider keys
disable related features. Anonymous telemetry is enabled unless disabled.

No deployment yet: NAS versus Netcup remains pending.
