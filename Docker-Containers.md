<!--
============================================================================
  DOCKER / CONTAINERS CHEAT SHEET (SKELETON)
  Companion to Windows-Admin-Cheat-Sheet.md and Linux-Mac-Admin-Cheat-Sheet.md
  Covers: Docker Engine / CLI  •  Docker Compose v2  •  Buildx  •  Registries
  Last Updated: 2026-07-22

  STATUS: SKELETON. This file is intentionally concise -- structure and the
  most load-bearing commands only, not an exhaustive reference (unlike the
  Windows and Linux/macOS sheets). Expand sections as needed; keep the
  numbered-section + ToC layout so it can later grow into its own
  single-page HTML web app, matching the other two sheets.

  SECURITY: NEVER store passwords, keys, registry tokens, or secrets in a
            Dockerfile, image layer, compose file, or shell history. Use
            `docker login` interactively, a secrets manager, Docker secrets
            (Swarm), or --env-file with a git-ignored file. Never bake
            credentials into `ENV`/`ARG` -- both persist in image history.

  COMPATIBILITY KEY:
    [ALL]     = Docker Engine / Docker Desktop on Linux, macOS, and Windows
    [LINUX]   = Linux host / systemd-managed dockerd specifically
    [DESKTOP] = Docker Desktop only (macOS/Windows GUI + VM backend)
    [DEP]     = Deprecated; shown with modern replacement
============================================================================
-->

# Docker & Containers Handbook (Skeleton) 2026

Concise starting-point reference for Docker Engine, the Docker CLI, Compose v2,
and Buildx. Companion to the [Windows](./Windows-Admin-Cheat-Sheet.md) and
[Linux/macOS](./Linux-Mac-Admin-Cheat-Sheet.md) admin cheat sheets in this
handbook. For host-level container/virtualization context (Podman, KVM/libvirt),
see the Linux/macOS sheet's Virtualization & Containers section.

> **Security:** Never hardcode registry credentials, API tokens, or secrets in
> a Dockerfile, compose file, or image layer -- both `ENV` and `ARG` persist in
> `docker history`. Use `docker login`, a secrets manager, or `--env-file`
> pointed at a git-ignored file.

**Compatibility key:** `[ALL]` Linux/macOS/Windows · `[LINUX]` Linux host/systemd
specifically · `[DESKTOP]` Docker Desktop only · `[DEP]` deprecated.

---

## Table of Contents

1.  [Install & Daemon](#1-install--daemon)
2.  [Images](#2-images)
3.  [Containers](#3-containers)
4.  [Dockerfile Basics](#4-dockerfile-basics)
5.  [Volumes & Bind Mounts](#5-volumes--bind-mounts)
6.  [Networking](#6-networking)
7.  [Docker Compose](#7-docker-compose)
8.  [Buildx & Multi-Arch](#8-buildx--multi-arch)
9.  [Registries](#9-registries)
10. [Logs, Stats & Debugging](#10-logs-stats--debugging)
11. [Cleanup & Disk Space](#11-cleanup--disk-space)
12. [Security Basics](#12-security-basics)
13. [Orchestration Beyond Compose](#13-orchestration-beyond-compose)

---

## [1] Install & Daemon

```bash
# [LINUX] status / start / enable at boot
systemctl status docker
sudo systemctl enable --now docker

# [ALL] version + environment info
docker version
docker info

# [LINUX] run docker without sudo (log out/in after)
sudo usermod -aG docker "$USER"

# [DESKTOP] macOS/Windows: install via Docker Desktop; CLI behaves the same
```

## [2] Images

```bash
docker pull nginx:latest
docker images
docker build -t myapp:1.0 .
docker tag myapp:1.0 myapp:latest
docker rmi myapp:1.0
docker history myapp:1.0          # inspect layers (never bake secrets here)
docker save -o myapp.tar myapp:1.0   # export; docker load -i myapp.tar to import
```

## [3] Containers

```bash
docker run -d --name web -p 8080:80 nginx
docker ps                 # running
docker ps -a               # all, including stopped
docker exec -it web sh     # shell into a running container
docker logs -f web
docker stop web; docker start web; docker restart web
docker rm -f web            # force remove (stops first)
docker inspect web | less   # full JSON config
```

## [4] Dockerfile Basics

```dockerfile
# Multi-stage build skeleton -- keeps final image small
FROM node:22-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:22-alpine
WORKDIR /app
COPY --from=build /app/dist ./dist
USER node                   # avoid running as root
EXPOSE 3000
CMD ["node", "dist/server.js"]
```

```bash
docker build --no-cache -t myapp:1.0 .
docker build --build-arg NODE_ENV=production -t myapp:1.0 .
```

## [5] Volumes & Bind Mounts

```bash
docker volume create mydata
docker volume ls
docker run -d -v mydata:/var/lib/data nginx     # named volume (managed by Docker)
docker run -d -v "$PWD":/app nginx              # bind mount (host path)
docker volume prune                              # remove unused volumes
```

## [6] Networking

```bash
docker network ls
docker network create mynet
docker run -d --network mynet --name db postgres
docker network inspect mynet
docker port web                                  # show published ports
```

## [7] Docker Compose

```bash
# Compose v2 is a `docker` subcommand (plugin), not a separate binary.
# [DEP] docker-compose (v1, standalone binary) -- migrate to `docker compose`.
docker compose up -d
docker compose down
docker compose ps
docker compose logs -f
docker compose exec web sh
docker compose build --no-cache
docker compose config          # validate + print resolved config
```

## [8] Buildx & Multi-Arch

```bash
docker buildx create --use
docker buildx build --platform linux/amd64,linux/arm64 -t myapp:1.0 --push .
docker buildx ls
```

## [9] Registries

```bash
docker login                                     # Docker Hub (or -u/--password-stdin)
docker login ghcr.io -u USERNAME --password-stdin < token.txt
docker tag myapp:1.0 ghcr.io/OWNER/myapp:1.0
docker push ghcr.io/OWNER/myapp:1.0
docker logout
```

## [10] Logs, Stats & Debugging

```bash
docker logs --tail 200 -f web
docker stats                     # live CPU/mem/net per container
docker top web                   # processes inside the container
docker events                    # live daemon event stream
```

## [11] Cleanup & Disk Space

```bash
docker system df                  # disk usage summary
docker system prune                # remove stopped containers, unused networks/images (dangling)
docker system prune -a --volumes   # aggressive -- ALSO removes unused images + volumes; use with caution
docker container prune
docker image prune -a
```

## [12] Security Basics

```bash
docker scout quickview myapp:1.0    # vulnerability overview (Docker Desktop/Hub account)
docker run --read-only nginx        # read-only root filesystem
docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE nginx   # minimal capabilities
docker run --pid=host --network=host ...   # AVOID unless required -- weakens isolation
```

## [13] Orchestration Beyond Compose

```bash
# Compose = single host. For multi-host orchestration:
# - Docker Swarm (built into the engine, lighter-weight):
docker swarm init
docker stack deploy -c docker-compose.yml mystack
docker service ls

# - Kubernetes (industry standard for larger deployments) -- out of scope for
#   this skeleton; see a dedicated Kubernetes reference when this section
#   is expanded.
```

---

**Docker & Containers Handbook (Skeleton) 2026** -- companion to the
[Windows](./Windows-Admin-Cheat-Sheet.md) and
[Linux/macOS](./Linux-Mac-Admin-Cheat-Sheet.md) sheets in this handbook.
Maintained by Chris Grady ([@cgfixit](https://github.com/cgfixit)) for
high-agency sysadmins. Source:
[cgfixit/Windows-Linux--Docker-Handbook](https://github.com/cgfixit/Windows-Linux--Docker-Handbook).
This is a skeleton -- contributions and expansion welcome.

<!-- END OF SKELETON -->
