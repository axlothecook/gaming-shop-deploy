# Gaming Shop — Deploy to Raspberry Pi (learning doc)

This is the **teaching companion** to `PLAN.md` Step 7. It explains *what* each
piece is, *why* it's there, and *what decision you're making* — so you can follow,
question, and correct the setup, not paste it blind.

Read it top to bottom once. Then execute the checklist at the end.

**Locked decisions (from the 2026-05-17 planning):**
- Runtime: **Docker Compose** on the Pi.
- Networking: **Cloudflare Tunnel** + one paid domain (~$10/yr), subdomain per site.
- CI: **GitHub Actions** → build images → push to **GHCR** → Pi pulls + restarts.
- Database: **MongoDB in a container on the Pi** + nightly `mongodump` backup, off-device.
- Storage: **Cloudflare R2** (migrated off Supabase 2026-05-27 — Supabase free tier pauses after 7 days idle, taking images offline). Served via `images.axlothecook.com`, shared bucket `axlothecook-images`, `gameshop/` key prefix.

---

## 0. The mental model — what "deploy" actually means here

Right now the app runs on your PC: you start MongoDB, `npm start` the backend,
`npm run dev` the frontend. "Deploying" means making that same thing run **on the
Pi, by itself, reachable from the internet, and restarting itself on a code push.**

Four problems to solve, each maps to one tool:

| Problem | Tool | Why |
|---|---|---|
| "Runs on my PC" → "runs identically on a Pi" | **Docker** | packages each app + its exact environment into an image that runs the same anywhere |
| "3 separate things to start" → "one command" | **Docker Compose** | declares all containers (backend, frontend, mongo, tunnel) in one file, starts them together |
| "Reachable from the internet" without exposing your home | **Cloudflare Tunnel** | the Pi dials *out* to Cloudflare; visitors hit Cloudflare, not your home IP |
| "I pushed code" → "the Pi updates itself" | **GitHub Actions** | on every push, builds new images and tells the Pi to pull + restart |

You don't need to master all four before starting — but you should understand
*why each exists*. The rest of this doc builds that understanding.

---

## 1. Docker — the concept

**The problem Docker solves:** "works on my machine." Your PC has a specific Node
version, OS, installed libraries. The Pi has different ones (ARM CPU, different OS).
Code that runs on one can break on the other. Docker eliminates that by shipping
the *environment together with the code*.

**Three words you must not confuse:**

- **Image** — a frozen, read-only snapshot: "Node 20 + your backend code + its
  npm packages." A blueprint. Built once, doesn't change.
- **Container** — a running instance of an image. The image is the recipe; the
  container is the cooked meal. You can run many containers from one image.
- **Dockerfile** — a text file of instructions for *building* an image: "start
  from Node 20, copy my code in, run `npm install`, the start command is `node app.js`."

Flow: you write a **Dockerfile** → `docker build` turns it into an **image** →
`docker run` turns the image into a running **container**.

**Why this matters for the Pi:** you build the image once (in GitHub Actions, or
on the Pi), and it runs identically regardless of the Pi's OS quirks. The Pi only
needs Docker installed — not Node, not specific libraries.

**One Pi-specific gotcha — CPU architecture.** Your PC is x86/amd64; the Pi is
**ARM (arm64)**. A Docker image is built *for* an architecture. An image built on
your x86 PC won't run on the ARM Pi unless you build it for ARM. Two ways to handle
it (decided in §6): build the image *on the Pi itself* (always native), or build a
multi-arch image in GitHub Actions. We'll use the simpler path first.

### What you'll write — two Dockerfiles

**Backend `Dockerfile`** (Express app). Conceptually:
1. Start from an official `node` base image (ARM-compatible — the official ones are multi-arch).
2. Copy `package.json` + `package-lock.json`, run `npm ci` (installs exact locked versions).
3. Copy the rest of the source.
4. Declare the start command (`node app.js`).
- *Why copy `package.json` first, separately from the code?* Docker caches each
  step ("layer"). If only your code changed but not your dependencies, Docker
  reuses the cached `npm ci` layer — much faster rebuilds. This ordering is a
  deliberate optimization, not arbitrary.

**Frontend `Dockerfile`** (SvelteKit). This one is a **multi-stage build**:
- *Stage 1 (build):* Node image, `npm ci`, `npm run build` — produces the compiled
  `build/` output.
- *Stage 2 (run):* a fresh smaller Node image, copy *only* the `build/` output +
  production deps from stage 1, start with `node build`.
- *Why two stages?* The build tools (Vite, the SCSS compiler, dev dependencies)
  are huge and only needed *during* the build. The final image ships only what's
  needed to *run* — smaller, faster, less attack surface. This is standard practice.
- ⚠️ **Reminder from the pre-flight audit:** the frontend must first switch
  `@sveltejs/adapter-auto` → `@sveltejs/adapter-node` (see `PLAN.md` Step 7
  verify-list). `adapter-node` is what produces a runnable `node build` server.
  Also: the frontend's `LINK` env var is baked in **at build time** — so the
  production `LINK` value must be passed into stage 1 of the build.

---

## 2. Docker Compose — the concept

One Dockerfile = one image = one container. But Gaming Shop is **four** containers:
backend, frontend, mongo, cloudflared. Starting/stopping/connecting them by hand is
tedious and error-prone.

**Docker Compose** is a single file (`docker-compose.yml`) that declares all of them
— what image each uses, what env vars, what ports, how they connect, what order —
and `docker compose up` starts the whole stack together.

**Key concepts in the compose file:**

- **Services** — each container is a "service" (`backend`, `frontend`, `mongo`,
  `cloudflared`).
- **The Docker network** — Compose puts all services on a private virtual network,
  and **they reach each other by service name**. So the backend connects to Mongo
  at `mongodb://mongo:27017/...` — `mongo` is literally the service name, not
  `localhost`. This is why your Mongo URI changes for the Pi: locally it's
  `localhost:27017`, in Compose it's `mongo:27017`.
- **Volumes** — containers are *ephemeral*: delete the container, its internal data
  is gone. A **volume** is persistent storage on the Pi's disk, mounted into a
  container. **MongoDB's data must live in a volume** — otherwise restarting the
  Mongo container wipes your database. This is the single most important Compose
  concept to get right.
- **Ports** — `"3000:3000"` maps a container port to a Pi port. Note: with
  Cloudflare Tunnel you mostly *don't* expose ports publicly — the tunnel reaches
  services over the internal network. Ports are only mapped if you need local access.
- **`depends_on`** — ordering: start `mongo` before `backend`.
- **`environment` / `env_file`** — how env vars get into a container (see §5).
- **`restart: unless-stopped`** — if a container crashes or the Pi reboots, Docker
  restarts it. This is your "always-on" guarantee — the Compose equivalent of pm2.

**What you'll have:** one `docker-compose.yml` (likely in a small separate "deploy"
repo, or the backend repo) describing all four services. `docker compose up -d` on
the Pi brings the whole site up.

---

## 3. Cloudflare Tunnel — the concept

Covered in planning, recap of the *why*: a normal server opens a port to the
internet (port-forwarding) — that exposes your home IP and network. **Cloudflare
Tunnel inverts it:** a small program (`cloudflared`) running on the Pi makes an
*outbound* connection to Cloudflare and holds it open. Visitors hit Cloudflare's
servers; Cloudflare passes the request down the tunnel to your Pi. No ports opened,
home IP hidden, TLS handled by Cloudflare, works behind CGNAT.

**As a container:** `cloudflared` runs as the 4th service in `docker-compose.yml`.
It's configured with a tunnel token (from the Cloudflare dashboard) and a mapping:
"hostname `gaming-shop.yourdomain.com` → the `frontend` service on the internal
network; `api.yourdomain.com` → the `backend` service."

**What you do once, in the Cloudflare dashboard:**
1. Add your domain to Cloudflare (change nameservers at your registrar).
2. Create a Tunnel — get a token.
3. Add public hostname routes (subdomain → internal service).

The tunnel routing by hostname is also what makes the **future archery domain
switch** trivial (see archery memory) and lets multiple sites share one Pi.

---

## 4. GitHub Actions + GHCR — the concept

**The problem:** after the first deploy, every code change would mean: SSH into the
Pi, pull, rebuild, restart — by hand, every time. **CI/CD automates that.**

**GitHub Actions** = automation that runs *on GitHub's servers* when something
happens in your repo (here: a push to `main`). You describe it in a YAML file in
`.github/workflows/`. That file is a **workflow**: a list of **jobs**, each a list
of **steps** (checkout code, build, etc.).

**GHCR (GitHub Container Registry)** = a free place to *store* Docker images,
attached to your GitHub account. Think "npm registry, but for Docker images."

**The pipeline, end to end:**
1. You `git push` to `main`.
2. GitHub Actions wakes up, on GitHub's servers:
   - checks out the code,
   - `docker build` the image(s) — **built for ARM** so the Pi can run them,
   - pushes the image(s) to GHCR.
3. The workflow then **connects to the Pi over SSH** and runs:
   `docker compose pull && docker compose up -d` — the Pi downloads the new image
   and restarts that service. Old container out, new one in.
4. Site is updated. Build logs are visible in GitHub's UI.

**Why build on GitHub, not the Pi?** The Pi's CPU is slow; building a SvelteKit
image on it takes a while. GitHub's runners are fast and free (for public repos /
within free minutes). The Pi just *pulls a finished image* — quick.

**Secrets:** the workflow needs the Pi's SSH key, the production env values, etc.
These go in **GitHub repo Secrets** (Settings → Secrets) — encrypted, never in the
code. The workflow references them as `${{ secrets.NAME }}`.

**What you'll write:** a `.github/workflows/deploy.yml` in each app repo (or one in
a deploy repo). Start simple — even a manual-trigger version first — then add the
push trigger.

---

## 5. Environment variables & secrets — where everything lives

This trips people up, so it gets its own section. **Three different places hold
config, for three different reasons:**

1. **Your local `.env` files** — for development on your PC. **Never committed**
   (gitignored). Stay exactly as they are.
2. **GitHub repo Secrets** — values the *build/deploy pipeline* needs: the Pi's SSH
   key, GHCR token, and any value baked in at **build time** (the frontend `LINK` —
   see §1). Set in GitHub's UI.
3. **On the Pi** — values the *running containers* need at runtime: `MONGO_URI`,
   `PORT`, the `R2_*` keys (Cloudflare R2 storage), the Cloudflare tunnel token
   (`TUNNEL_TOKEN`). These live in a `.env` file **on the Pi** (not in any repo)
   that `docker-compose.prod.yml` reads via `env_file`.

⚠️ Recall the dotenv lesson: env vars injected into a container land in
`process.env` — there's no `.env` *file* inside the container unless you put one
there. The backend code already reads `process.env.X` everywhere (after the
`gamesControllers.js` fix), so it's fine. Just remember: **`.env` files are never
baked into images** — config is injected at run time (or, for the one build-time
case, passed as a build arg).

---

## 6. Architecture-decision notes (the ARM question)

The one genuinely fiddly Docker topic for a Pi. Two viable paths:

- **(A) Build the image on the Pi.** The workflow SSHes in and runs
  `docker compose up -d --build` — the Pi builds natively for its own ARM CPU.
  Simple, no architecture mismatch ever. Downside: the Pi does the (slow) build.
- **(B) Build multi-arch in GitHub Actions** using `docker buildx` with
  `--platform linux/arm64`, push to GHCR, Pi just pulls. Fast, but more workflow
  complexity (buildx, QEMU emulation).

**Recommendation for learning:** start with **(A)** — fewer moving parts, you see
the build happen, easier to debug. Move to (B) later as an optimization once the
pipeline works and you understand it. (This is a real decision point — when we get
here you can push back.)

---

## 7. Suggested learning order

You don't need all four tools at once. Learn and apply in this order — each builds
on the last, and each stage is independently testable:

1. **Docker basics** — install Docker on your PC, run an existing public image
   (`docker run hello-world`), understand image vs container. ~1 short session.
2. **Write the two Dockerfiles** — backend, then frontend. Build them locally on
   your PC, run the containers, confirm the apps work *in containers* before any Pi
   involvement.
3. **Docker Compose** — write `docker-compose.yml`, get all 4 services running
   together *on your PC*. This is the whole app, containerized, before the Pi.
4. **The Pi** — install 64-bit Pi OS + Docker on the Pi, copy the compose stack
   over, `docker compose up`. Site runs on the Pi (LAN-only at this point).
5. **Cloudflare Tunnel** — add the domain, the tunnel, the `cloudflared` service.
   Site is now public.
6. **GitHub Actions** — automate it. Start with a manual-trigger workflow, then add
   the push-to-`main` trigger.
7. **Backup cron** — the nightly `mongodump` (see §8).

Each numbered stage is a natural "stop, verify, ask questions" point.

---

## 8. The MongoDB backup

Decided: Mongo runs in a container; its data is in a Docker **volume** on the Pi.
SD cards fail — so a backup is mandatory.

- A **cron job on the Pi** runs nightly (e.g. 3 AM): `mongodump --archive --gzip`
  against the mongo container → one compressed archive file.
- The script copies that archive **off the Pi** (to a target — for Gaming Shop, the
  user's PC is acceptable; for archery it must be a cloud bucket / NAS — see archery
  memory).
- **Rotation:** keep the last ~14 archives, delete older ones, on both ends. This
  caps total storage at a fixed size (≈ 14 × one DB dump) — it does NOT grow
  unboundedly. An unchanged DB → identical small dumps → flat footprint.

---

## 9. Pre-deploy checklist (the actual to-do)

Concept understanding first (above); then execute:

- [ ] **Pi prep** — 64-bit Pi OS installed; Docker + Docker Compose installed; SSH enabled; note the Pi's LAN IP.
- [ ] **Frontend code change** — `svelte.config.js`: `adapter-auto` → `adapter-node`; `npm i -D @sveltejs/adapter-node`. (PLAN.md Step 7 verify-item.)
- [ ] **Backend code change** — tighten `app.js` `cors()` to `cors({ origin: <frontend URL from env> })`, env-driven. (PLAN.md Step 7 verify-item.)
- [ ] **Backend `Dockerfile`** — write + test build locally.
- [ ] **Frontend `Dockerfile`** — multi-stage; write + test build locally; confirm `LINK` build arg wired.
- [ ] **`docker-compose.yml`** — 4 services (backend, frontend, mongo, cloudflared); mongo data on a named volume; `restart: unless-stopped`; `env_file` for the Pi `.env`.
- [ ] **Run the whole stack on your PC** — `docker compose up` locally, verify the site works fully containerized.
- [ ] **Domain** — register (~$10/yr), add to Cloudflare, nameservers pointed.
- [ ] **Cloudflare Tunnel** — create tunnel, get token, add hostname routes (frontend + backend subdomains).
- [ ] **Deploy to the Pi** — copy the compose stack + a Pi-side `.env`; `docker compose up -d`; verify the site is reachable via the domain.
- [ ] **GitHub Actions** — `.github/workflows/deploy.yml`; add repo Secrets (SSH key, env values); start manual-trigger, then push trigger.
- [ ] **Backup cron** — nightly `mongodump --archive --gzip`, keep-last-14 rotation, copy off-device.
- [ ] **Final verify (PLAN.md Step 7 list)** — `adapter-node` active, CORS tightened, frontend `LINK` present in the CI build.
