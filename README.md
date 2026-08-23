# flaresolverr

Shared FlareSolverr sidecar for Cloudflare challenge solving. Standalone-usable with plain
Docker Compose, or deployed as a shared fleet service on Coolify for any tenant app that
needs to scrape Cloudflare-protected pages (currently `job-digest`).

## Standalone Usage

```bash
docker compose up -d
```

- **API endpoint:** http://localhost:8191/v1
- **Health check:** http://localhost:8191/health

## On Coolify

- Deployed with `docker_compose_location` set to `/docker-compose.coolify.yaml`.
- Joins the `coolify` external network.
- Never gets a public domain (treated as shared infrastructure like `browser`, `postgresql`,
  `qdrant`, or `valkey`).

## Using from Another App

Point `FLARESOLVERR_URL` to `http://flaresolverr:8191`. No authentication is enabled on
this port (refer to `CLAUDE.md` for accepted-risk reasoning).

### Example request

```bash
curl -s -X POST http://flaresolverr:8191/v1 \
  -H "Content-Type: application/json" \
  -d '{"cmd":"request.get","url":"https://example.com","maxTimeout":60000}'
```

## What FlareSolverr does (and doesn't) solve

FlareSolverr solves **Cloudflare JS/managed challenges** — the "Just a moment..."
interstitial pages that block headless HTTP clients. It does **not** help with:

- Hard IP-reputation blocks that aren't a challenge to solve (e.g., Indeed's "Additional
  Verification Required")
- Other anti-bot vendors (e.g., DataDome on Monster.ca)
- Unbranded verification walls that aren't Cloudflare underneath

## Image

Uses `ghcr.io/flaresolverr/flaresolverr:latest`. The `latest` tag is tracked by Renovate
(weekly check) which opens PRs when the upstream image digest changes.

## License

GPL-3.0 — see [LICENSE](LICENSE).
