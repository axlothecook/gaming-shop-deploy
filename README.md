# Gaming shop deploy
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

## Related repositories
[Front end](https://github.com/axlothecook/Gaming-shop-frontend): the SvelteKit site this stack serves <br />
[Back end](https://github.com/axlothecook/Gaming-Shop): the Express + MongoDB API <br />
[Umbrella](https://github.com/axlothecook/gameshop): a joint repo for all Gaming Shop related repositories
