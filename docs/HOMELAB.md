# Homelab deployment

Support Fins serves static files with Nginx on container port 80. Geometry processing
runs in the browser. No database, GPU, upload directory, or persistent volume is needed.
GitHub Actions publishes this fork to ghcr.io/prinako/support-fins for amd64 and arm64.
Pushes to main publish latest, main, and a sha tag. Version tags such as v1.2.3 publish
1.2.3 and 1.2 tags; they do not replace latest. Pull requests build without publishing.
The image is linked to this repository through its OCI source label.

After the first successful publication, check the package visibility in GitHub Packages.
New packages may be private; for anonymous homelab pulls set the package to Public.
For a private package, authenticate the Docker host with docker login ghcr.io using a
classic PAT with read:packages (or configure equivalent registry credentials in Arcane).

## Start on the Docker host

Clone this repository on the host, then run from its root:

```bash
git clone https://github.com/prinako/support-fins.git
cd support-fins
docker network inspect traefik-net
docker compose -f docker-compose.homelab.yml config
docker compose -f docker-compose.homelab.yml up -d
docker compose -f docker-compose.homelab.yml ps
curl -f http://127.0.0.1:8731/
```

Use this Compose file alone, rather than merging it with docker-compose.yml.
It expects the existing external bridge network traefik-net. If that network is missing,
choose an existing shared bridge network using SUPPORT_FINS_NETWORK. Only create a new
one if needed; attach your proxy to the same network.

Open http://YOUR_SERVER_IP:8731/ from the LAN. The published port binds all host interfaces
by default. Optional values in a .env file beside the Compose file:

```dotenv
SUPPORT_FINS_VERSION=latest
SUPPORT_FINS_NETWORK=traefik-net
SUPPORT_FINS_PORT=8731
SUPPORT_FINS_BIND_IP=0.0.0.0
```

Set SUPPORT_FINS_BIND_IP to the server's LAN address to bind only that address.
This setup uses a shared Docker bridge network; it does not assign a macvlan address.

## Traefik

The container joins the shared network but this file does not create a hostname or router.
To enable routing with Traefik's Docker provider, add labels under services.web in
docker-compose.homelab.yml. Replace the hostname, entrypoint, and certificate resolver
with the values configured in your Traefik instance:

```yaml
    labels:
      - "traefik.enable=true"
      - "traefik.docker.network=traefik-net"
      - "traefik.http.routers.support-fins.rule=Host(`fins.example.com`)"
      - "traefik.http.routers.support-fins.entrypoints=websecure"
      - "traefik.http.routers.support-fins.tls=true"
      - "traefik.http.routers.support-fins.tls.certresolver=YOUR_RESOLVER"
      - "traefik.http.routers.support-fins.service=support-fins"
      - "traefik.http.services.support-fins.loadbalancer.server.port=80"
```

Match traefik.docker.network to SUPPORT_FINS_NETWORK if you change it.
Traefik must join that same network. Point local DNS for the chosen hostname at
Traefik's LAN address. Once proxy access works, the ports section can be removed
if direct LAN access is unnecessary.

## Cloudflare Tunnel

If cloudflared shares this Docker network, set its service URL to
http://support-fins:80. If cloudflared runs elsewhere, use the Docker host's LAN
address and published port, such as http://YOUR_SERVER_IP:8731.
Use a dedicated hostname with the application at /; subpath hosting is not configured.
Use either this direct tunnel origin or a route through Traefik to match your setup.

## Arcane

Paste docker-compose.homelab.yml into an Arcane Compose project. It pulls the published
image directly, so no Git checkout or local build context is needed. Ensure the external
network exists and configure registry credentials if the package is private.

## Update

Recreate the service to pull the current selected tag. Set SUPPORT_FINS_VERSION to an
existing version tag to pin a release.

```bash
git pull --ff-only
docker compose -f docker-compose.homelab.yml up -d
docker compose -f docker-compose.homelab.yml ps
```

No server deployment is performed by adding these repository files.
