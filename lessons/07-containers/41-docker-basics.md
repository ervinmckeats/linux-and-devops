# Lesson 41. Docker basics

## Goal
Run, inspect, debug and clean up containers, and understand images and layers.

## Theory

- An **image** is a read-only template: a stack of filesystem **layers** + metadata (command, env, ports). Built once, run anywhere with a compatible kernel.
- A **container** is a running process created from an image, with namespaces and cgroups (lesson 38) and a thin writable layer on top. Delete the container → the writable layer is gone.
- A **registry** stores images: Docker Hub, GitHub Container Registry (ghcr.io), AWS ECR, GitLab, Harbor.
- Image name: `registry/namespace/name:tag`, for example `ghcr.io/org/api:1.4.2`. `latest` is an ordinary tag. It does not mean "the newest". Pin versions, or *better*, digests (`@sha256:...`).

Containers vs VMs: a VM has its own kernel and boots an OS; a container shares the host kernel and starts in milliseconds. Weaker isolation, much lighter.

**Docker** (daemon + CLI) and **Podman** (daemonless, rootless by default, same CLI: `alias docker=podman`) both run OCI images. Kubernetes uses **containerd** or **CRI-O** directly.

## Install

```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker $USER
docker version; docker info
```

- `curl -fsSL https://get.docker.com -o get-docker.sh`: read it first
- `sudo usermod -aG docker $USER`: then log out/in. Note: docker group = root-equivalent

## Run

```bash
docker run hello-world
docker run -it --rm ubuntu:24.04 bash
docker run -d --name web -p 8080:80 nginx:1.27
docker run -d --name web2 -p 127.0.0.1:8081:80 nginx:1.27
docker run -d -e POSTGRES_PASSWORD=secret -v pgdata:/var/lib/postgresql/data postgres:16
docker run --rm -v "$PWD":/work -w /work python:3.12-slim python script.py
docker run -d --restart unless-stopped --memory 512m --cpus 1 myapp:1.0
```

- `docker run -it --rm ubuntu:24.04 bash`: interactive, remove on exit
- `docker run -d --name web -p 8080:80 nginx:1.27`: background, publish port host:container
- `docker run -d --name web2 -p 127.0.0.1:8081:80 nginx:1.27`: only localhost (bypasses ufw otherwise, lesson 26)

## Inspect and debug

```bash
docker ps
docker ps -a
docker logs -f --tail 100 web
docker exec -it web sh
docker inspect web
docker inspect -f '{{.State.ExitCode}} {{.State.OOMKilled}}' web
docker stats
docker top web
docker port web
docker events
docker cp web:/etc/nginx/nginx.conf .
```

- `docker ps`: running
- `docker ps -a`: all, including exited
- `docker exec -it web sh`: shell inside a running container
- `docker inspect web`: everything: IP, mounts, env, restart count, exit code
- `docker stats`: live CPU/memory per container
- `docker top web`: processes inside
- `docker events`: what the daemon is doing

Debugging a container that exits immediately:

```bash
docker ps -a
docker logs web
docker run -it --entrypoint sh myapp:1.0
```

- `docker ps -a`: see exit code
- `docker logs web`: last output
- `docker run -it --entrypoint sh myapp:1.0`: start a shell instead of the app

Exit codes: `0` normal, `1` app error, `125` docker run failed, `126` not executable, `127` command not found, `137` SIGKILL (often OOM: check `OOMKilled`), `143` SIGTERM.

## Images

```bash
docker images
docker pull nginx:1.27
docker history nginx:1.27
docker image inspect nginx:1.27 --format '{{.Config.Cmd}} {{.Config.ExposedPorts}}'
docker tag myapp:1.0 ghcr.io/org/myapp:1.0
docker login ghcr.io
docker push ghcr.io/org/myapp:1.0
docker save myapp:1.0 | gzip > myapp.tar.gz
docker load < myapp.tar.gz
```

- `docker history nginx:1.27`: layers and their size
- `docker save myapp:1.0 | gzip > myapp.tar.gz`: move without a registry

## Lifecycle and cleanup

```bash
docker stop web
docker start web; docker restart web
docker rm -f web
docker system df
docker container prune
docker image prune -a
docker volume prune
docker system prune
```

- `docker stop web`: SIGTERM, then SIGKILL after 10 s
- `docker system df`: space used by images, containers, volumes, cache
- `docker image prune -a`: remove unused images
- `docker volume prune`: careful: deletes data in unused volumes

A stopped container keeps its writable layer. A growing `/var/lib/docker` is one of the most common "disk full" causes. Also configure log rotation for the json-file driver:

`/etc/docker/daemon.json` (then `sudo systemctl restart docker`; applies to new containers):

```json
{
  "log-driver": "json-file",
  "log-opts": { "max-size": "50m", "max-file": "3" }
}
```

## Signals and PID 1

Inside the container your app is PID 1. PID 1 does not get default signal handlers, so an app that does not handle `SIGTERM` will be killed after the timeout instead of shutting down cleanly. Use the exec form of `CMD` (lesson 42) and handle SIGTERM in the app, or run with `--init` (tiny init that forwards signals and reaps zombies).

## Practice

1. Run nginx on port 8080, change `index.html` with `docker cp` and `docker exec`, check with curl.
2. Run a container that is OOM-killed: `docker run --memory 50m python:3.12-slim python -c 'a=" "*10**9'`. Read `State.OOMKilled` and the exit code.
3. Run PostgreSQL with a named volume, create a table, delete the container, start a new one with the same volume, check the table is there.
4. Find the biggest layer of `python:3.12` with `docker history`. Compare with `python:3.12-slim` and `python:3.12-alpine`.
5. Fill disk with exited containers and dangling images, then clean up with `docker system df` / `prune`.

## Check yourself

- Image vs container?
- Why is `latest` a bad tag for deployments?
- What does exit code 137 with `OOMKilled=true` mean?
- Why is being in the `docker` group equivalent to root?

## Next
[Lesson 42. Dockerfile best practices](42-dockerfile.md)
