# Deploy stack for Gaming Shop repo
This repo runs the whole Gaming Shop on Raspberry Pi. It consists of config files that describe how to run the 4 Docker containers together: docker-compose.prod.yml for the Pi, docker-compose.dev.yml for local development, and .env.example.
<br />
<br />

## Docker containers
<ul> 
	<li>database: MongoDB</li> 
	<li>backend: Express API (from GHCR)</li> 
	<li>frontend: the public site (from GHCR)</li> 
	<li>cloudflared: Cloudflare Tunnel</li> 
</ul>
<br />


## What does each container do
### [MongoDB](https://www.mongodb.com)
It stores games, genres and developers as documents, along with the links to their R2 images. Its data lives in a named volume, so a redeploy does not wipe it, and it restarts on its own if it crashes.

### The backend 
It runs the API image pulled from GHCR, reads its config and secrets from the .env file, and waits for the database container to start before it does.

### The frontend 
It runs the site image pulled from GHCR. It is the only container visitors ever reach; it talks to the backend by its service name inside the Docker network, so the backend never needs a public address.

### [Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-tunnel/)
It runs Cloudflare's tunnel client, which dials out to Cloudflare. That is how the [site's domain](https://gameshop.axlothecook.com) reaches the frontend without port forwarding or a static IP on my home network.

### The dev stack
This is docker-compose.dev.yml: the same idea minus the tunnel. It builds the images from the sibling repos on my PC instead of pulling them from GHCR.
<br />
<br />

## Why no graph here
The runtime graph for this stack already lives in the [umbrella README](https://github.com/axlothecook/gameshop/blob/main/README.md). It shows exactly the containers this repo runs and how they connect, so this README does not repeat it.
<br />
<br />

## The config
Since I cannot commit .env to git, and the real values live only in the Pi's .env, the .env.example lists what the Pi needs: the Mongo connection values, the R2 keys for image storage, and the Cloudflare TUNNEL_TOKEN.
<br />
<br />

## Backups
A cron job on the Pi runs a nightly mongodump (compressed archive) and keeps the last 5 days of archives; each one also gets copied off the Pi to my PC, so an SD card failure cannot take the data with it. The backup script lives on the Pi itself, not in this repo.
<br />
<br />

## Deployment
Handled by my shared [CI/CD pipeline](https://github.com/axlothecook/homelab-ci-cd):
(1) a push to the frontend repo runs its tests and 
(2) CI builds the arm64 image
(3) then the Pi pulls it and restarts the stack. 

The backend repo's CI is build-only, so a backend change goes live with the next stack restart.
<br />
<br />

## Tech stack
[Docker Compose](https://docs.docker.com/compose/): runs the 4 containers together as one stack. There is also a second file, `docker-compose.dev.yml`, for local development: it builds the images on my PC from the sibling repos instead of pulling them from GHCR <br />
[GitHub Actions](https://github.com/features/actions) + GHCR (arm64): CI builds the images and GHCR stores them <br />
[Tailscale](https://tailscale.com): the private connection CI uses to reach the Pi <br />
[Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-tunnel/): connects the Pi to the internet without opening ports, and sends visitors straight to the frontend container. This project has no nginx in front; the frontend's own Node server answers the requests <br />
[mongodump](https://www.mongodb.com/docs/database-tools/mongodump/): the nightly database backups
<br />

## Where this sits in the pipeline's evolution
This is the first version of my [CI/CD pipeline](https://github.com/axlothecook/homelab-ci-cd). The deploy job did not exist at first: for the first week, CI only built the image and I updated the Pi by hand. The backend repo still works that way, so a backend change goes live with the next stack restart.

Compared to the later versions, this one is missing a few things. Fir starters old images are not cleaned up after deploys, the frontend publishes a port on the Pi, and there is no reverse proxy. The frontend's test gate also came much later; I brought it back here after building the gates for the archery project.
