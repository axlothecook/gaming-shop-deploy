# gaming-shop-deploy

Deployment orchestration for the Gaming Shop project — ties together the
backend (`Gaming-Shop`), frontend (`Gaming-shop-frontend`), and MongoDB.

## Local development run

1. `cp .env.example .env` and fill in the Supabase values (from the backend's `.env`).
2. `docker compose up --build`
3. Open http://localhost:3000

## Files

- `docker-compose.yml` — the local-dev stack (3 services; builds images from the
  sibling repos via relative paths). Production/Pi will use a separate
  compose file pulling pre-built images from GHCR.
- `.env.example` — template for the backend env vars; copy to `.env` (gitignored).
