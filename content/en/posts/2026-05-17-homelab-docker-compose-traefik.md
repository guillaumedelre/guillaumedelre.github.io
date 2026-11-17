---
title: "Building a self-hosted homelab with Docker Compose and Traefik"
aliases: ["/2026/02/17/building-a-self-hosted-homelab-with-docker-compose-and-traefik/"]
date: 2026-05-17
lastmod: 2026-11-17
categories: [devops]
tags: [homelab, docker, traefik]
description: "Complete guide to setting up a Docker homelab with Traefik and sslip.io: independent stacks, auto-configured dashboard, common pitfalls documented."
---

For years I wanted a homelab at home. A place of my own to host development tools, monitor my machines, run home automation, and experiment without risking breaking anything important. The idea is simple. Getting it running, a bit less so.

Back then, Kubernetes didn't exist yet. Options for running multiple services on a single machine came down to bash scripting, hand-written Nginx configs, and a lot of coffee. Tutorials on "homelab for humans" were nowhere to be found.

This tutorial is what I wish I had found back then. It's been running for several years now. Not without evolving: services added, others dropped, choices revisited. But the foundation is there, stable — and that's what success looks like in self-hosting.

The setup: ten self-hosted services on a local machine, accessible from a browser via readable URLs, without touching DNS configuration, without renting a VPS, without managing TLS certificates. The ingredient that makes it possible: [sslip.io][sslip], a public DNS service that encodes the IP directly in the domain name. `service.192.168.1.10.sslip.io` resolves to `192.168.1.10`, with zero configuration, from any machine on the local network.

This tutorial is aimed at someone who knows Docker but is starting from scratch on self-hosted service orchestration.

---

## Table of contents

1. [Philosophy and architecture choices](#1-philosophy-and-architecture-choices)
2. [The building blocks](#2-the-building-blocks)
3. [Step-by-step setup](#3-step-by-step-setup)
4. [Adding a new service](#4-adding-a-new-service)
5. [Patterns and conventions](#5-patterns-and-conventions)
6. [Common pitfalls](#6-common-pitfalls)
7. [Conclusion](#conclusion)
8. [References](#references)

---

## 1. Philosophy and architecture choices

### Goal

Run multiple web services on a local machine, accessible from a browser via readable URLs, without touching DNS configuration, without renting a VPS, without managing TLS certificates.

### Why Docker Compose and not something else?

Docker Compose is the right level of complexity for a personal homelab. Kubernetes is too heavy for a single machine. Docker Swarm is in decline. Compose is simple, readable, versionable, and sufficient for dozens of services.

### Why Traefik and not Nginx Proxy Manager?

**Nginx Proxy Manager (NPM)** is a graphical interface for configuring Nginx as a reverse proxy. Routes are stored in a database and configured through a UI.

**[Traefik][traefik]** automatically reads Docker container labels and generates its configuration on the fly. When a container starts with the right labels, Traefik discovers it and creates the route immediately, without restarting, without opening any UI.

This "configuration as code" approach has two major advantages:
- A service's configuration lives in its `compose.yaml`, in the same place as everything else.
- Adding a service requires no changes to Traefik.

### Why Dockge and not Portainer?

**Portainer** is a full Docker management tool: images, volumes, networks, individual containers... powerful but complex.

**[Dockge][dockge]** is focused on a single thing: managing Docker Compose stacks. Its UI is minimal and intuitive. For a homelab where everything is managed through Compose, it's sufficient and much more pleasant to use.

### Why sslip.io?

Web services need a hostname (e.g. `dozzle.myserver.local`) for Traefik to route correctly. The usual options:
- Edit `/etc/hosts` on every machine: tedious, not shareable.
- Set up a local DNS server (Pi-hole, AdGuard): requires additional infrastructure.
- Buy a domain and configure DNS: costs money and time.

**sslip.io** is a public DNS service that automatically resolves `<anything>.<IP>.sslip.io` to `<IP>`. Example: `dozzle.192.168.1.10.sslip.io` resolves to `192.168.1.10`. Nothing to configure — the DNS works everywhere without touching anything.

---

## 2. The building blocks

### The shared Docker network

All services and Traefik must share the same Docker network so Traefik can communicate with them. This network is called `traefik` and is created once:

```bash
docker network create traefik
```

It is an **external** network (created outside any Compose file). Each `compose.yaml` declares it as external:

```yaml
networks:
    traefik:
        external: true
```

Why external rather than internal to a Compose file? Because multiple independent stacks all need to connect to it. A network internal to a Compose file is only accessible to services within that file.

### Traefik: the reverse proxy

Traefik listens on port 80 and routes HTTP requests to the right container based on the `Host` header. Its `compose.yaml` also publishes 443 and 25565, but no entrypoint uses them yet: they are reserved for later.

Its main configuration lives in `stacks/traefik/docker/traefik/traefik.yaml`:

```yaml
api:
    dashboard: true
    insecure: true
    debug: true

entryPoints:
    web:
        address: :80
        forwardedHeaders:
            insecure: true
    ping:
        address: ":8082"

ping:
    entryPoint: ping

providers:
    docker:
        endpoint: unix:///var/run/docker.sock
        exposedByDefault: false

global:
    sendAnonymousUsage: false

log:
    level: WARN
```

`exposedByDefault: false` is important: Traefik ignores all containers by default. A container must explicitly opt in with the label `traefik.enable: true`. This prevents accidentally exposing services.

The `ping` entrypoint on port 8082 is dedicated to health checks, and the top-level `ping` block is what actually serves `/ping` on it. Without that block, the endpoint doesn't exist, the health check fails and Traefik's own container stays `unhealthy`. Keeping it off the `web` entrypoint also keeps health check requests out of the access logs.

To access the Docker daemon, Traefik mounts the socket:

```yaml
volumes:
    - /var/run/docker.sock:/var/run/docker.sock
```

### Dockge: the stack manager

Dockge runs inside a container itself (the `compose.yaml` at the root of the repo). It needs two things:
1. Access to the Docker socket to manage the other containers.
2. Access to the stack directories to read and edit `compose.yaml` files.

The critical point is the stack mount. Dockge launches stacks by passing absolute paths to the Docker daemon. These paths must be identical inside the Dockge container and on the host. The solution:

```yaml
volumes:
    - ${PWD}/stacks:${PWD}/stacks
environment:
    DOCKGE_STACKS_DIR: ${PWD}/stacks
```

`${PWD}` is a shell variable resolved at `docker compose up` time. It equals the current directory. If Dockge is launched from `/home/user/homelab`, the stacks folder will be mounted at `/home/user/homelab/stacks` on both sides. This is the only way to prevent Docker from creating ghost directories in the wrong place.

**Practical consequence**: always run `docker compose up -d` from the root of the repo.

Dockge's persistent data (configuration, history) lives in a named volume created in advance:

```bash
docker volume create homelab_dockge_data
```

Because it is declared `external: true`, `docker compose down -v` leaves it alone. A volume created by Compose itself, named or anonymous, would be removed with the stack.

---

## 3. Step-by-step setup

### Step 1: clone and configure

```bash
git clone <repo> homelab
cd homelab
```

Find the machine's local IP:

```bash
hostname -I | awk '{print $1}'
# e.g.: 192.168.1.10
```

Create and edit the root `.env`:

```bash
cp .env.example .env
# Edit .env:
# IP=192.168.1.10
# DOMAIN=sslip.io
# COMPOSE_PROJECT_NAME=dockge  ← important, see conventions section
```

### Step 2: Docker prerequisites

```bash
docker network create traefik
docker volume create homelab_dockge_data
```

### Step 3: start Dockge

```bash
docker compose up -d
```

Dockge is accessible at `http://<IP>:5001`. It is exposed directly on port 5001, not through Traefik (Traefik is not running yet at this point). Create an admin account on first launch.

### Step 4: configure the stacks

For each directory in `stacks/`, copy the `.env.example` (Watchtower has none, it needs no variable):

```bash
for stack in stacks/*/; do
    [ -f "${stack}.env.example" ] && cp "${stack}.env.example" "${stack}.env"
done
```

Then edit each `.env` to set `IP` and `DOMAIN` to the same values as in step 1. Most of them contain nothing else. `filebrowser` and `uptime-kuma` also set `COMPOSE_PROJECT_NAME` to their folder name: leave it as is (see conventions section).

For `filebrowser`, also set `FILEBROWSER_ROOT` to the local path to expose.

### Step 5: start the stacks from Dockge

From the Dockge interface (`http://<IP>:5001`), in this order:

**1. Traefik first**

Traefik must be running before the other services. Without Traefik, routes don't exist and services are unreachable via their URL.

After starting, verify Traefik is healthy:

```bash
docker ps --filter name=traefik
```

**2. The other stacks in any order**

Each stack automatically registers itself with Traefik via its Docker labels. Traefik discovers new containers in real time.

Homepage doesn't need any particular position: it queries the Docker labels of running containers each time the dashboard loads, so a service started after it shows up on the next refresh.

---

## 4. Adding a new service

Here is the `compose.yaml` template for any new service:

```yaml
services:
    myservice:
        image: vendor/myservice:latest
        restart: unless-stopped
        healthcheck:
            test: ["CMD-SHELL", "wget -qO- http://127.0.0.1:<PORT>/ || exit 1"]
            interval: 30s
            timeout: 10s
            retries: 3
            start_period: 10s
        labels:
            # Homepage - auto-discovery in dashboard
            homepage.group: outils
            homepage.name: My Service
            homepage.icon: https://cdn.jsdelivr.net/gh/selfhst/icons/webp/myservice.webp
            homepage.href: http://${COMPOSE_PROJECT_NAME}.${IP}.${DOMAIN}

            # Traefik - HTTP routing
            traefik.enable: true
            traefik.http.routers.myservice.entrypoints: web
            traefik.http.routers.myservice.rule: Host(`${COMPOSE_PROJECT_NAME}.${IP}.${DOMAIN}`)
            traefik.http.services.myservice.loadbalancer.server.port: <PORT>
        networks:
            - traefik

networks:
    traefik:
        external: true
```

And the associated `.env.example`:

```
IP=127.0.0.1
DOMAIN=sslip.io
```

**The folder name determines the subdomain.** If the folder is called `myservice`, the service will be accessible at `myservice.<IP>.<DOMAIN>`. That's it.

To find services worth adding, [selfh.st][selfhst] is an excellent resource: it's a catalog of self-hosted software organized by category (media, security, productivity, monitoring...), with a description, screenshot, and GitHub link for each. The site also publishes a weekly newsletter on new releases.

### Checklist for a new service

- [ ] Create `stacks/<subdomain-name>/compose.yaml`
- [ ] Create `stacks/<subdomain-name>/.env.example` with `IP` and `DOMAIN`
- [ ] Copy `.env.example` to `.env` and fill in IP/DOMAIN
- [ ] Check the port in the Traefik labels
- [ ] Choose the Homepage group: `infra`, `observabilité`, `outils`
- [ ] Find the icon on [selfhst/icons][selfhst-icons]
- [ ] Add persistent data in a volume if needed
- [ ] Start from Dockge and verify the container is `healthy`

---

## 5. Patterns and conventions

### The `${COMPOSE_PROJECT_NAME}` variable

Docker Compose automatically sets `COMPOSE_PROJECT_NAME` to the name of the folder holding the `compose.yaml`, which here is the stack folder. We use it to build URLs dynamically:

```yaml
traefik.http.routers.dozzle.rule: Host(`${COMPOSE_PROJECT_NAME}.${IP}.${DOMAIN}`)
homepage.href: http://${COMPOSE_PROJECT_NAME}.${IP}.${DOMAIN}
```

Advantage: no `*_HOST` variable to maintain in each `.env`. Renaming the folder automatically changes the subdomain.

**The exception is the root.** The repo folder is called `homelab`, but Dockge should answer on `dockge.<IP>.<DOMAIN>`. That's why the root `.env` sets `COMPOSE_PROJECT_NAME=dockge` explicitly. Stacks don't need it; the two that set it (`filebrowser`, `uptime-kuma`) just repeat their folder name.

### Homepage groups

Services are organized into three groups in the dashboard. The group names are in French in the repo (`observabilité` is monitoring, `outils` is tools):

| Group | Services |
|---|---|
| `infra` | [Traefik][traefik], [Dockge][dockge], [Watchtower][watchtower], [Homepage][homepage] |
| `observabilité` | [Dozzle][dozzle], [Glances][glances], [Uptime Kuma][uptime-kuma] |
| `outils` | [FileBrowser][filebrowser], [IT-Tools][ittools], [Stirling PDF][stirling-pdf] |

This grouping is specific to this homelab, not an enforced convention. Homepage accepts any value for `homepage.group`: you can create as many groups as needed and name them however you like (`media`, `home-automation`, `dev`...). The dashboard reorganizes automatically.

### Health checks

All services have a health check. This is crucial because **Traefik silently ignores `unhealthy` containers**: a service with a failing health check will not appear in routing, even with `traefik.enable: true`.

Three edge cases encountered in practice:

**1. `localhost` does not always mean `127.0.0.1`**

In some images, `localhost` resolves to the IPv6 address `::1` first, while the application only listens on IPv4. The health check then fails for no visible reason. Use `127.0.0.1` explicitly:

```yaml
test: ["CMD-SHELL", "wget -qO- http://127.0.0.1:8080/ || exit 1"]
```

**2. Images without a shell (`scratch`-based)**

Images based on `scratch` (e.g. Dozzle) do not contain `/bin/sh`. `CMD-SHELL` fails. Use the embedded binary:

```yaml
test: ["CMD", "/dozzle", "healthcheck"]
```

**3. Images without `wget` or `curl`**

Some Node.js or JVM images have neither wget nor curl. Possible solutions:
- If Node.js is available: `node -e "require('http').get('http://localhost:PORT', r => process.exit(r.statusCode < 400 ? 0 : 1)).on('error', () => process.exit(1))"`
- If curl is available: `curl -fs http://127.0.0.1:PORT/`
- If the app binary exposes a healthcheck subcommand: use it directly.

### Data persistence

For services that have data (configuration, user accounts, database):

```yaml
volumes:
    - ./docker/data:/path/in/container
```

The `./docker/` folder lives inside the stack directory and can be versioned, except for runtime data which goes in `.gitignore`.

**Rule**: add `stacks/<service>/docker/` to `.gitignore` if the folder contains data that should not be committed (SQLite databases, uploads...).

### Traefik label conventions

By convention, the name used in Traefik labels (`traefik.http.routers.<name>`) matches the Docker service name in `compose.yaml`. In practice, align it with the folder name:

```
stacks/it-tools/    →    service: ittools    →    traefik.http.routers.ittools.*
```

This is not a technical constraint from Traefik, just a readability convention.

---

## 6. Common pitfalls

### Dockge: Stop then Start, not Restart

When a `compose.yaml` is modified from an IDE and the changes need to be applied, use **Stop + Start** from Dockge, not "Restart". Restart restarts the existing container without re-reading the `compose.yaml`. Stop + Start recreates the container with the new configuration.

### Modified labels don't show up in Homepage

Homepage reads Docker labels on every page load, so a changed `homepage.group` or `homepage.name` should appear after a refresh. If it doesn't, the container still carries the old labels: it was restarted, not recreated (see Stop + Start above).

### Container starts but is not routable

Check in order:

1. `docker ps`: is the container `healthy`? Traefik ignores `unhealthy` containers.
2. Is the container on the `traefik` network?

```bash
docker inspect <container> --format '{{json .NetworkSettings.Networks}}'
```

3. Is the label `traefik.enable: true` present?
4. Does the `Host(...)` rule match the URL being tested?

### Mounting a file that doesn't exist yet

When a bind mount points to a **file** that does not yet exist on the host, Docker creates a **directory** in its place. It happens with Docker Engine as well as Docker Desktop on WSL. This ghost directory then blocks the mount of the actual file. Symptom: the container fails to start with a mount error.

Solution: ensure the file exists on the host before starting the container, or use a directory mount instead of a file mount.

### Watchtower: Docker API too old

On some configurations, Watchtower tries to communicate with the daemon starting the negotiation at API v1.25 (its historical minimum). Recent versions of Docker reject this version. Symptom: the container restarts in a loop with `client version 1.25 is too old. Minimum supported API version is 1.40`.

The repo's Watchtower `compose.yaml` forces the version:

```yaml
environment:
    DOCKER_API_VERSION: "1.40"
```

The value to use is the minimum stated in the error message, not your exact version. `1.40` was enough when the repo was written, but [Docker 29 raised the minimum to 1.44][watchtower-api]: on Docker 29 and later, set `1.44`. To check the actual API version of your daemon:

```bash
docker version --format '{{.Server.APIVersion}}'
```

Note that the Watchtower repository [was archived][watchtower] in December 2025: the image still works, but it no longer receives fixes. Keep that in mind before relying on it for automatic updates.

### `${PWD}` in Dockge's compose file

`${PWD}` is not a `.env` variable — it is a shell variable resolved at `docker compose up` time. It equals the current terminal directory. Running `docker compose up -d` from any other directory will produce a wrong value and break stack volume mounts.

---

*This homelab is designed to run on a Linux machine or WSL. All commands have been tested on Ubuntu/WSL2 with Docker Desktop.*

---

## Conclusion

I'm well aware this tutorial doesn't cover everything. We could have added authentication in front of each service, run the whole thing over HTTPS, set up a socket proxy to limit the Docker daemon's exposure, or pinned precise image versions. But each of those points would have considerably lengthened the article and the complexity of the setup. The goal was to start with something functional and maintainable, not to build a fortress on day one.

The perfect homelab doesn't exist. The one that runs, does.

<a class="gh-card" href="https://github.com/guillaumedelre/homelab" target="_blank" rel="noopener noreferrer">
  <span class="gh-card__head">
    <svg width="20" height="20" viewBox="0 0 24 24" aria-hidden="true"><path d="M12 .297c-6.63 0-12 5.373-12 12 0 5.303 3.438 9.8 8.205 11.385.6.113.82-.258.82-.577 0-.285-.01-1.04-.015-2.04-3.338.724-4.042-1.61-4.042-1.61C4.422 18.07 3.633 17.7 3.633 17.7c-1.087-.744.084-.729.084-.729 1.205.084 1.838 1.236 1.838 1.236 1.07 1.835 2.809 1.305 3.495.998.108-.776.417-1.305.76-1.605-2.665-.3-5.466-1.332-5.466-5.93 0-1.31.465-2.38 1.235-3.22-.135-.303-.54-1.523.105-3.176 0 0 1.005-.322 3.3 1.23.96-.267 1.98-.399 3-.405 1.02.006 2.04.138 3 .405 2.28-1.552 3.285-1.23 3.285-1.23.645 1.653.24 2.873.12 3.176.765.84 1.23 1.91 1.23 3.22 0 4.61-2.805 5.625-5.475 5.92.42.36.81 1.096.81 2.22 0 1.606-.015 2.896-.015 3.286 0 .315.21.69.825.57C20.565 22.092 24 17.592 24 12.297c0-6.627-5.373-12-12-12"/></svg>
    <span class="gh-card__repo">guillaumedelre/homelab</span>
  </span>
  <span class="gh-card__desc">Docker Compose homelab with Traefik — independent stacks, auto-configured dashboard, and zero DNS configuration using sslip.io.</span>
</a>

## References

| Project | Link |
|---|---|
| sslip.io | [sslip.io][sslip] |
| selfh.st | [selfh.st][selfhst] |
| Traefik | [github.com/traefik/traefik][traefik] |
| Dockge | [github.com/louislam/dockge][dockge] |
| Homepage | [github.com/gethomepage/homepage][homepage] |
| Dozzle | [github.com/amir20/dozzle][dozzle] |
| Glances | [github.com/nicolargo/glances][glances] |
| FileBrowser | [github.com/gtsteffaniak/filebrowser][filebrowser] |
| IT-Tools | [github.com/CorentinTh/it-tools][ittools] |
| Stirling PDF | [github.com/Stirling-Tools/Stirling-PDF][stirling-pdf] |
| Uptime Kuma | [github.com/louislam/uptime-kuma][uptime-kuma] |
| Watchtower | [github.com/containrrr/watchtower][watchtower] |
| selfhst/icons | [github.com/selfhst/icons][selfhst-icons] |

[sslip]: https://sslip.io
[selfhst]: https://selfh.st
[traefik]: https://github.com/traefik/traefik
[dockge]: https://github.com/louislam/dockge
[homepage]: https://github.com/gethomepage/homepage
[dozzle]: https://github.com/amir20/dozzle
[glances]: https://github.com/nicolargo/glances
[filebrowser]: https://github.com/gtsteffaniak/filebrowser
[ittools]: https://github.com/CorentinTh/it-tools
[stirling-pdf]: https://github.com/Stirling-Tools/Stirling-PDF
[uptime-kuma]: https://github.com/louislam/uptime-kuma
[watchtower]: https://github.com/containrrr/watchtower
[selfhst-icons]: https://github.com/selfhst/icons
[watchtower-api]: https://github.com/containrrr/watchtower/issues/2124
