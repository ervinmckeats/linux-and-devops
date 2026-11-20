# Lesson 43. Docker Compose, networks, volumes

## Goal
Run a multi-container application (app + database + cache + proxy) with persistent data and private networking.

## Networks

| Driver | Use |
|---|---|
| `bridge` (default) | containers on one host; NAT to the outside (like the namespaces lab in lesson 28) |
| user-defined bridge | **use this**: built-in DNS — containers reach each other by name |
| `host` | no network isolation; container uses host's interfaces (fast, no port mapping) |
| `none` | no network |
| `overlay` | across hosts (Swarm); in Kubernetes the CNI plugin does this |

```bash
docker network create appnet
docker run -d --name db --network appnet -e POSTGRES_PASSWORD=pw postgres:16
docker run --rm --network appnet postgres:16 pg_isready -h db     # "db" resolves by name
docker network inspect appnet
```

Containers on the default `bridge` network cannot resolve each other by name. Create your own network.

Only publish (`-p`) what must be reachable from outside. Databases usually need no published port at all.

## Storage

| Type | Syntax | Use |
|---|---|---|
| named volume | `-v pgdata:/var/lib/postgresql/data` | database data; managed by Docker in `/var/lib/docker/volumes` |
| bind mount | `-v /srv/conf:/etc/app:ro` | config files from the host, source code in dev |
| tmpfs | `--tmpfs /tmp` | scratch data in RAM |

Back up a named volume:

```bash
docker run --rm -v pgdata:/data -v "$PWD":/backup alpine tar -czf /backup/pgdata.tgz -C /data .
```

For databases prefer logical dumps (`pg_dump`, lesson 21).

Bind mount permission problems: the container's UID must be allowed to read/write the host path. Check with `docker exec c id` and `ls -ln` on the host.

## Docker Compose

Describe the whole stack in one file, `compose.yaml`:

```yaml
services:
  proxy:
    image: nginx:1.27
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf:ro
    depends_on:
      app:
        condition: service_healthy
    restart: unless-stopped

  app:
    build: ./app
    image: ghcr.io/example/app:1.0
    environment:
      DATABASE_URL: postgres://app:${DB_PASSWORD}@db:5432/app
      REDIS_URL: redis://cache:6379/0
    healthcheck:
      test: ["CMD", "python", "-c", "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')"]
      interval: 10s
      timeout: 3s
      retries: 5
    depends_on:
      db:
        condition: service_healthy
      cache:
        condition: service_started
    deploy:
      resources:
        limits:
          memory: 512M
          cpus: "1.0"
    restart: unless-stopped

  db:
    image: postgres:16
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB: app
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app"]
      interval: 5s
      retries: 10
    restart: unless-stopped

  cache:
    image: redis:7
    command: ["redis-server", "--maxmemory", "128mb", "--maxmemory-policy", "allkeys-lru"]
    restart: unless-stopped

volumes:
  pgdata:
```

`.env` next to the file (not in Git):

```
DB_PASSWORD=change-me
```

Compose creates a network for the project automatically, so `app` reaches `db` and `cache` by service name.

```bash
docker compose up -d            # create and start everything
docker compose ps
docker compose logs -f app
docker compose exec db psql -U app
docker compose build app && docker compose up -d app   # rebuild and replace one service
docker compose pull && docker compose up -d            # update images
docker compose config           # final config with variables resolved - check before up
docker compose down             # stop and remove containers and network (volumes stay)
docker compose down -v          # ALSO delete volumes = delete data
```

### Overrides for dev and prod

```bash
# compose.yaml (shared) + compose.override.yaml (dev, loaded automatically)
docker compose -f compose.yaml -f compose.prod.yaml up -d
```

Dev override might bind-mount the source code and enable debug; prod pins image tags and adds resource limits.

### Profiles

```yaml
  adminer:
    image: adminer
    profiles: ["debug"]
```
`docker compose --profile debug up -d` starts it only when asked.

## Compose in production?

Fine for a single server: small projects, internal tools, a self-hosted monitoring stack. Run it from a systemd unit or with `restart: unless-stopped`. When you need many hosts, rolling updates without downtime, autoscaling and self-healing across machines, move to Kubernetes (Module 9).

## Practice

1. Write a compose file for nginx + a small app + PostgreSQL + Redis. App connects to DB by name.
2. Add health checks and make the app wait for the DB with `condition: service_healthy`.
3. Stop the stack with `down`, start it again: data persists. Then `down -v`: data is gone.
4. Back up the DB volume, delete it, restore it.
5. Use `docker compose config` to find a typo in a variable.
6. Run the stack as a systemd service that starts on boot.

## Check yourself

- Why can containers on a user-defined network find each other by name?
- Difference between a named volume and a bind mount?
- What does `docker compose down -v` delete?
- Does `depends_on` alone wait until the database accepts connections?

## Next
Module 8 — DevOps: lesson 44, CI/CD.
