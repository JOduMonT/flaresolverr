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

## Security hardening — simpler than `browser` in one way, same in another

Unlike the `browser` sidecar (which needs `cap_add: [CHOWN, FOWNER, DAC_OVERRIDE, SETUID,
SETGID]` for its s6-overlay privilege-drop entrypoint), FlareSolverr is a plain Python HTTP
server. `cap_drop: ALL` works clean with zero `cap_add`.

**Deliberately no `read_only: true`** — despite being "just" a Python HTTP server,
FlareSolverr bundles undetected_chromedriver which writes to `/app/.local` at startup
(`Patcher.__init__` → `os.makedirs(self.data_path)`), and Chrome itself needs additional
write paths. Confirmed by CI smoke test failure: `OSError: [Errno 30] Read-only file system:
'/app/.local'`. Same posture and same reasoning as the `browser` sidecar: write surface too
broad to safely enumerate without another crash-loop discovery cycle.

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
