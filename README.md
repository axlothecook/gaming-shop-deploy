# gaming-shop-deploy

Deployment orchestration for the Gaming Shop project — ties together the
backend (`Gaming-Shop`), frontend (`Gaming-shop-frontend`), MongoDB, and the
Cloudflare Tunnel.

**Live:** <https://gameshop.axlothecook.com> (self-hosted on a Raspberry Pi,
exposed via Cloudflare Tunnel — no open ports / no static IP).

## Related repositories

- [Front end](https://github.com/axlothecook/Gaming-shop-frontend.git)
- [Back end](https://github.com/axlothecook/Gaming-Shop)

## Stacks

- **`docker-compose.dev.yml`** — local-dev stack (3 services: mongo, backend,
  frontend; builds images from the sibling repos via relative paths).
- **`docker-compose.prod.yml`** — production/Pi stack (4 services: mongo,
  backend, frontend, **cloudflared**); pulls pre-built `arm64` images from GHCR
  rather than building on the Pi.

## Local development run

1. `cp .env.example .env` and fill in the values (see below).
2. `docker compose -f docker-compose.dev.yml up --build`
3. Open http://localhost:3000

## Environment (`.env`, gitignored)

Copy `.env.example` and fill in:

- **Mongo** — `NODE_ENV_DB_LOCALHOST`, `NODE_ENV_PORT_LOCALHOST`, collection paths.
- **Cloudflare R2** (image storage) — `R2_ACCOUNT_ID`, `R2_BUCKET`,
  `R2_PUBLIC_BASE`, `R2_ACCESS_KEY_ID`, `R2_SECRET_ACCESS_KEY`. Images are served
  from `images.axlothecook.com` (shared bucket `axlothecook-images`,
  `gameshop/` key prefix). Migrated off Supabase, which pauses free-tier
  projects after 7 days idle.
- **Cloudflare Tunnel** (prod only) — `TUNNEL_TOKEN` from the Zero Trust
  dashboard tunnel (`axlothecook-pi`). Routes `gameshop.axlothecook.com` ->
  `frontend:3000`.

## Production deploy (the Pi)

CI handles it: pushing to `Gaming-shop-frontend` main builds the image and runs
a `deploy-to-pi` job that SSHes to the Pi over Tailscale and runs
`docker compose -f docker-compose.prod.yml pull && up -d`. The backend repo's
CI is build-only; deploying a backend change is a manual `pull` + `up -d` on the
Pi (or piggybacks on the next frontend deploy).

Manual run on the Pi (from `~/gaming-shop-deploy`):

```sh
docker compose -f docker-compose.prod.yml pull
docker compose -f docker-compose.prod.yml up -d
```
