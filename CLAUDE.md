# CLAUDE.md — flaresolverr

Shared Cloudflare challenge-solving sidecar. Read `README.md` first for the
standalone-vs-Coolify basics — this file is the "don't repeat past mistakes" layer.

## This is shared infrastructure, not an app

Deployed once per tenant, reused by every app that needs Cloudflare challenge solving. It
must **never** get a public domain or a published host port — same convention as this fleet's
shared `browser`/Postgres/Qdrant/Valkey. When creating the Coolify application, explicitly
suppress the auto-assigned domain: `PATCH /applications/<uuid>` with
`{"docker_compose_domains": [{"name":"flaresolverr","domain":""}]}`.

## The API port is 8191, unauthenticated

FlareSolverr's HTTP API (`POST /v1`) has no authentication — no password, no token, nothing.
Once this service is on the shared `coolify` network, *any* other container already on that
network can submit URLs for challenge solving. Docker network membership is the only boundary
today. Accepted as a bounded risk: every container in this fleet is first-party and curated,
not arbitrary/untrusted code. Revisit if that ever changes.

## Security hardening — simpler than `browser`

Unlike the `browser` sidecar (which needs `cap_add: [CHOWN, FOWNER, DAC_OVERRIDE, SETUID,
SETGID]` for its s6-overlay privilege-drop entrypoint), FlareSolverr is a plain Python HTTP
server. `cap_drop: ALL` works clean with zero `cap_add`.

`read_only: true` is also safe here — FlareSolverr is stateless. Chrome's internal write
needs go to `/tmp`, which is tmpfs-mounted. Same reasoning does NOT apply to the `browser`
sidecar, which runs a full desktop environment with a write surface too broad to enumerate.

## The `latest` tag is Renovate-managed, not careless

FlareSolverr publishes only a `latest` tag (no versioned tags). Renovate tracks the actual
image digest and opens PRs when it changes — same pattern as every other "latest-only" image
in this fleet. The `check-release.yml` workflow runs weekly to trigger this.

## Known gap: no CAPTCHA solver configured

FlareSolverr supports pluggable CAPTCHA solvers via `CAPTCHA_SOLVER`, but none is configured
here. The built-in Cloudflare challenge solver handles JS/managed challenges without one. If
a future source requires hCaptcha or reCAPTCHA solving, that's a new env var + config, not
a structural change.

## Migration from bare `docker run`

This container was previously deployed as a one-off `docker run --restart unless-stopped` on
the Coolify host (2026-08-22). Promoting it to a proper Coolify-managed Application (this
repo) gives it the same lifecycle as `browser`: CI smoke tests, Renovate tracking, version
history, and consistent security hardening.
