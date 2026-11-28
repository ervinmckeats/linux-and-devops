# Lesson 42. Dockerfile best practices

## Goal
Build images that are small, fast to rebuild, reproducible and safe.

## How builds work

Each instruction (`FROM`, `RUN`, `COPY`...) creates a layer. Docker **caches** layers: if an instruction and its inputs did not change, the cached layer is reused. Once one layer changes, **every layer after it** is rebuilt.

Order instructions from "changes rarely" to "changes often".

## Bad vs good

Bad:

```dockerfile
FROM python:latest
COPY . /app
RUN pip install -r /app/requirements.txt
CMD python /app/app.py
```

Problems: unpinned base, any code change reinstalls all dependencies, runs as root, shell form CMD (no signals), huge image, build context may include `.git` and secrets.

Good:

```dockerfile
# syntax=docker/dockerfile:1
FROM python:3.12-slim AS base
ENV PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1
WORKDIR /app

# dependencies first: this layer is cached until requirements change
COPY requirements.txt .
RUN --mount=type=cache,target=/root/.cache/pip pip install -r requirements.txt

# then the code
COPY src/ ./src/

RUN useradd --system --uid 10001 app
USER 10001

EXPOSE 8000
HEALTHCHECK --interval=30s --timeout=3s CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')"
# exec form: the app is PID 1 and receives SIGTERM
CMD ["python", "-m", "src.app"]
```

## .dockerignore

Keeps the build context small and secrets out:

```
.git
.env
*.log
node_modules
__pycache__
.venv
tests/
Dockerfile
```

## Multi-stage builds

Build with all the tools, ship only the result.

```dockerfile
# syntax=docker/dockerfile:1
FROM golang:1.23 AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -ldflags="-s -w" -o /out/server ./cmd/server

FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=build /out/server /server
USER nonroot
ENTRYPOINT ["/server"]
```

Result: a few MB, no shell, no package manager, nothing for an attacker to use.

Node.js example:

```dockerfile
FROM node:22-slim AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --omit=dev

FROM node:22-slim
WORKDIR /app
ENV NODE_ENV=production
COPY --from=deps /app/node_modules ./node_modules
COPY . .
USER node
CMD ["node", "server.js"]
```

## Rules of thumb

| Rule | Why |
|---|---|
| Pin base image versions (or digests) | reproducible builds |
| Small base: `-slim`, distroless, `alpine` | less size, fewer CVEs. Alpine uses musl: some Python/C deps behave differently |
| Combine `apt-get update && apt-get install` in one `RUN`, clean lists | no stale cache layer, smaller image |
| `--no-install-recommends` | fewer packages |
| Dependencies before code | cache hits |
| Never put secrets in `ENV`, `ARG` or `COPY` | they stay in layers and `docker history` forever |
| Use `RUN --mount=type=secret` for build secrets | not stored in the image |
| Non-root `USER` | limits damage on escape |
| Exec form `CMD ["app"]` | correct signals |
| One process per container | simple lifecycle, logs, scaling |
| Log to stdout/stderr | the runtime collects logs |

```dockerfile
RUN apt-get update \
 && apt-get install -y --no-install-recommends curl ca-certificates \
 && rm -rf /var/lib/apt/lists/*
```

Build secret example:

```bash
docker build --secret id=npmrc,src=$HOME/.npmrc .
# in Dockerfile: RUN --mount=type=secret,id=npmrc,target=/root/.npmrc npm ci
```

## ENTRYPOINT vs CMD

- `ENTRYPOINT`: the executable. `CMD`: default arguments. `docker run image ARGS` replaces `CMD`.
- Common pattern: `ENTRYPOINT ["/entrypoint.sh"]` that prepares config and ends with `exec "$@"`, plus `CMD ["app"]`. The `exec` matters: it replaces the shell so the app gets signals.

## Build, tag, scan

```bash
docker build -t myapp:1.0 .

# stop at a stage
docker build --target build -t myapp:build .

docker buildx build --platform linux/amd64,linux/arm64 -t ghcr.io/org/myapp:1.0 --push .
docker image ls myapp

# vulnerabilities
trivy image myapp:1.0

docker scout cves myapp:1.0

# explore layers, find waste
dive myapp:1.0

# lint
hadolint Dockerfile
```

Tag with the version **and** the Git commit: `myapp:1.4.2`, `myapp:sha-3f2c1ab`. That links a running container to exact code.

## Run it safely

```bash
docker run --read-only --tmpfs /tmp \
  --cap-drop ALL --security-opt no-new-privileges \
  --user 10001 --memory 256m --cpus 0.5 \
  -p 127.0.0.1:8000:8000 myapp:1.0
```

## Practice

1. Write a Dockerfile for the Python log tool from lesson 32 (or any small web app). Build it.
2. Change one line of code and rebuild. Which layers were cached? Reorder the Dockerfile until only the last layer rebuilds.
3. Build a Go or Node app with a multi-stage build. Compare the single-stage and multi-stage image sizes.
4. Put a fake secret in `ENV`, build, and find it with `docker history --no-trunc`. Then fix it with a build secret.
5. Scan your image with `trivy` and reduce the number of HIGH/CRITICAL findings by changing the base image.
6. Run your image with `--read-only --cap-drop ALL` and fix whatever breaks.

## Check yourself

- Why copy `requirements.txt` before the source code?
- Why is `ENV API_KEY=...` in a Dockerfile a leak even if you delete it in a later layer?
- What is the benefit of distroless images, and what do you lose?

---

<p align="left">
  <a href="media/42-dockerfile.mp4"><img width="640" src="media/42-dockerfile.gif" alt="Lesson 42 animation: Dockerfile best practices"></a>
</p>

## Next
[Lesson 43. Docker Compose, networks, volumes](43-compose-networks-volumes.md)
