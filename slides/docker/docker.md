---
title: Docker
---

![Three containers on one host](/assets/images/topics/docker.svg)
<!-- .element: class="title-illustration" -->

# Docker

Build once, run anywhere — Linux containers for Python apps.

---

## Why containers?

- **One artifact** — same image runs on your laptop, CI, and prod
- **No "works on my machine"** — environment is in the image
- **Isolation** — processes, filesystem, network are namespaced
- **Reproducible** — `docker pull image:1.2.3` is the same bytes everywhere
- **Composable** — `docker compose up` brings up your DB, cache, app together

For deployment, containers are the modern default.

---

## Install

```bash
# macOS / Windows
brew install --cask docker          # or download Docker Desktop

# Linux (Ubuntu/Debian)
curl -fsSL https://get.docker.com | sh

docker --version
# Docker version 27.x

docker run hello-world
# Hello from Docker!
```

---

## Image, container, registry

| Term | What it is |
| --- | --- |
| **Image** | A read-only snapshot — code + dependencies + OS layer |
| **Container** | A running instance of an image (process + writable layer) |
| **Registry** | Where images live remotely (Docker Hub, GHCR, ECR) |
| **Tag** | A label on an image (`python:3.13-slim`, `myapp:1.2.3`) |

You **build** images, **run** containers, **push/pull** to/from registries.

---

## Dockerfile — minimal Python

```dockerfile
FROM python:3.13-slim

WORKDIR /app

COPY pyproject.toml uv.lock ./
RUN pip install uv && uv sync --frozen --no-dev

COPY . .

CMD ["uv", "run", "python", "main.py"]
```

```bash
docker build -t my-app:0.1 .
# Step 1/5 : FROM python:3.13-slim
#  ...
# Successfully tagged my-app:0.1
```

That's a working container for a small Python app.

---

## Dockerfile — common instructions

| Instruction | What it does |
| --- | --- |
| `FROM` | Base image to start from |
| `WORKDIR` | `cd` for subsequent commands; creates if missing |
| `COPY src dst` | Copy from build context into the image |
| `ADD` | Like `COPY` but with URL & tar handling — prefer `COPY` |
| `RUN cmd` | Run a shell command at build time, commit the result |
| `ENV K=V` | Set an environment variable |
| `EXPOSE 8000` | Document a port (doesn't actually publish it) |
| `CMD ["python", "main.py"]` | Default command when running the container |
| `ENTRYPOINT` | Like CMD, harder to override — use sparingly |

---

## Running a container

```bash
docker run --rm my-app:0.1                      # run, remove on exit
docker run -d --name api my-app:0.1             # detached (background)
docker run -p 8000:8000 my-app:0.1              # publish ports
docker run -e DATABASE_URL=... my-app:0.1       # env var
docker run -v ./data:/data my-app:0.1           # mount a local dir

docker ps                                        # running containers
docker logs api                                  # last logs
docker logs -f api                               # follow live
docker exec -it api bash                         # shell into a container
docker stop api && docker rm api
```

`-it` = interactive + TTY. `-d` = detached.

---

## Layered caching — make builds fast

Docker caches each `RUN` / `COPY` layer. Order matters:

```dockerfile
# BAD — every code change re-runs `uv sync`
COPY . .
RUN uv sync

# GOOD — deps cached separately from code
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev
COPY . .
```

If `pyproject.toml` and `uv.lock` haven't changed, the dep install layer is reused — code-only changes rebuild in seconds.

---

## Multi-stage builds — smaller images

Compile in one stage, copy artifacts to a slimmer runtime:

```dockerfile
FROM python:3.13 AS builder
WORKDIR /app
COPY pyproject.toml uv.lock ./
RUN pip install uv && uv sync --frozen --no-dev

FROM python:3.13-slim AS runtime
WORKDIR /app
COPY --from=builder /app/.venv /app/.venv
COPY . .
ENV PATH="/app/.venv/bin:$PATH"
CMD ["python", "main.py"]
```

The final image only has the venv + your code, not build tools or `uv` itself.

---

## `docker compose` — multi-service apps

Define services in YAML, bring them up together:

```yaml
# compose.yml
services:
  api:
    build: .
    ports: ["8000:8000"]
    environment:
      DATABASE_URL: postgresql://app:app@db:5432/app
    depends_on: [db, redis]
  db:
    image: postgres:17
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: app
    volumes: [pgdata:/var/lib/postgresql/data]
  redis:
    image: redis:7

volumes: { pgdata: }
```

--

## Compose — daily commands

```bash
docker compose up                  # foreground, all services
docker compose up -d               # detached
docker compose up --build          # rebuild images
docker compose ps                  # status
docker compose logs -f api         # follow one service
docker compose exec api bash       # shell into a running container
docker compose down                # stop + remove containers
docker compose down -v             # also delete volumes
```

Service names work as DNS — your `api` service can reach `db:5432` directly.

---

## .dockerignore

Same idea as `.gitignore`, but for the build context (what gets sent to Docker):

```
.venv/
__pycache__/
*.pyc
.git/
.pytest_cache/
.mypy_cache/
.ruff_cache/
.env
.env.local
node_modules/
```

Without this, your build sends gigabytes of caches to the daemon and ends up in the image. Always commit a `.dockerignore`.

---

## Best practices for Python images

- **Use `python:X.Y-slim`** — smaller than full, has glibc/openssl, runs almost anything
- **Pin base image patch versions** — `python:3.13.1-slim`, not `python:3.13-slim`, for byte-for-byte reproducibility
- **Run as non-root** — `RUN useradd app && USER app`
- **Don't `COPY .` first** — it busts the cache. Copy lock files, install deps, then code
- **Multi-stage builds for native deps** — `gcc` in the builder, only the wheels in runtime
- **Health check** — `HEALTHCHECK CMD curl -f http://localhost:8000/health || exit 1`

---

## Image size matters

```bash
docker images
# REPOSITORY   TAG     SIZE
# python       3.13            1.0 GB
# python       3.13-slim       150 MB
# python       3.13-alpine     60 MB     ← gotcha: musl libc
# my-app       0.1             185 MB
```

`alpine` is small but uses musl libc — wheels compiled for glibc may not work, leading to source builds and slower images. Stick to `slim` unless you have a reason.

---

## Tagging and registries

```bash
docker tag my-app:0.1 ghcr.io/me/my-app:0.1
docker login ghcr.io
docker push ghcr.io/me/my-app:0.1

# On a server / in CI:
docker pull ghcr.io/me/my-app:0.1
docker run --rm ghcr.io/me/my-app:0.1
```

GitHub Container Registry (`ghcr.io`) is free for open repos and integrates with GitHub Actions. AWS ECR, GCP Artifact Registry, Docker Hub all work the same way.

---

## What's next

- **Ansible** — provisioning the host that runs your containers
- **Deployment** — wiring CI → registry → server
