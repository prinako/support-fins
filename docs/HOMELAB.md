# Homelab deployment

Support Fins serves static files with Nginx on container port 80. Geometry processing
runs in the browser. No database, GPU, upload directory, or persistent volume is needed.
The existing Dockerfile and nginx.conf can be used unchanged.

The original Compose file references ghcr.io/gittrahan/support-fins. The fork's current
container workflow builds but does not publish images. This deployment builds the checked-out
source into the local image support-fins:homelab instead.

## Start on the Docker host

Clone this repository on the host, then run from its root:

```bash
git clone https://github.com/prinako/support-fins.git
cd support-fins
docker network inspect traefik-net
docker compose -f docker-compose.homelab.yml config
docker compose -f docker-compose.homelab.yml up -d --build
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

Use a Git-backed project checkout of this repository and select
docker-compose.homelab.yml if your Arcane installation supports choosing the Compose file.
The build context must include Dockerfile, nginx.conf, LICENSE, and web/.
Pasting only the YAML into an empty project is insufficient for a local build.
Otherwise run the commands above on the Docker host.

## Update

```bash
git pull --ff-only
docker compose -f docker-compose.homelab.yml up -d --build
docker compose -f docker-compose.homelab.yml ps
```

No server deployment is performed by adding these repository files.
