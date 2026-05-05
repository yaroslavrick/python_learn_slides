---
title: Deployment
---

![Deploy pipeline: push, build, registry, deploy, health](/assets/images/topics/deployment.svg)
<!-- .element: class="title-illustration" -->

# Deployment

CI → registry → server. The shape of a Python deploy in 2026.

---

## What "deploy" means

The work between **"green CI on `main`"** and **"new code is serving requests"**:

1. Build an artifact (image / wheel / static bundle)
2. Push it somewhere reachable
3. Tell the runtime (server / orchestrator / serverless platform) to use the new version
4. Verify it's healthy
5. Roll back if not

The substance changes per stack; the shape doesn't.

---

## A modern Python deploy stack

```
push to main
     │
     ↓
GitHub Actions
  ├─ build image (Docker)
  ├─ push to ghcr.io
  └─ deploy step:
       ├─ Fly.io / Render / Railway       (managed)
       ├─ Kubernetes (helm / kustomize)   (orchestrated)
       ├─ Docker on a VPS                 (single host)
       └─ AWS ECS / GCP Cloud Run         (cloud-managed containers)
```

We'll cover the simplest end of this — single-host Docker — then sketch the rest.

---

## Container deploy — single host

```yaml
# .github/workflows/deploy.yml
name: Deploy
on: { push: { branches: [main] } }

jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions: { packages: write }
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
```

Checkout the repo, set up buildx, log into GitHub Container Registry. The next step does the actual build + push.

--

## Container deploy — build and push

```yaml
      - uses: docker/build-push-action@v6
        with:
          push: true
          tags: |
            ghcr.io/${{ github.repository }}:latest
            ghcr.io/${{ github.repository }}:${{ github.sha }}
```

Two tags every commit: SHA-tagged for rollbacks, `latest` for "what's serving now".

--

## Container deploy — pull on the server

Continuing the workflow with an SSH step that pulls the new image:

```yaml
      - name: Pull and restart
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.PROD_HOST }}
          username: deploy
          key: ${{ secrets.PROD_SSH_KEY }}
          script: |
            cd /srv/app
            docker compose pull
            docker compose up -d
            docker image prune -f
```

`docker compose up -d` recreates only changed services. `--no-deps` speeds it up if you only changed the app and not the DB.

---

## Migrations as part of deploy

```yaml
      - name: Migrate, then deploy
        run: |
          ssh deploy@$HOST "
            cd /srv/app &&
            docker compose run --rm api uv run python manage.py migrate &&
            docker compose pull api &&
            docker compose up -d api
          "
```

Important rules:

- Migrations must be **forward- and backward-compatible** during the rollout window
- Don't mix schema changes and code changes that depend on each other in one deploy — split into two: (1) deploy schema; (2) deploy code that uses it
- `--rm` so the migration container exits and doesn't leave a corpse

---

## Health checks and rollback

```python
# In your app:
@app.get("/health")
async def health():
    # Check DB, cache, anything that means "I can serve"
    return {"status": "ok"}
```

Rollback when the health check stays red:

```bash
docker compose pull api
docker compose up -d api
sleep 10
curl -fsS http://localhost:8000/health || (
  docker compose tag api:previous api:current   # actually:
  docker compose down api
  docker run -d ... ghcr.io/me/api:<previous-sha>
)
```

In real life, a load balancer or platform (Fly, Render, ECS, K8s) does this for you. Don't write the rollback script yourself if you can avoid it.

---

## Static files & CDN

For Django / FastAPI sites with assets:

```bash
uv run python manage.py collectstatic --noinput
```

Push `staticfiles/` to S3 / Cloudflare R2 / a CDN, point Django at the URL with `STATIC_URL`. Don't serve static files from your app server — the CDN is faster and cheaper.

For SPA frontends, the bundle (Vite/webpack output) goes to the same CDN.

---

## Environment variables

Never bake secrets into images. Pass them at runtime:

```yaml
# compose.yml
services:
  api:
    image: ghcr.io/me/app:latest
    env_file: /etc/app.env       # ← secrets live here, root-only
```

`/etc/app.env`:

```
DATABASE_URL=postgresql://...
SECRET_KEY=...
SENTRY_DSN=...
```

Provision this file with **Ansible** (`copy` module, `mode: 0600`, owned by `root`). For richer setups, use **Vault** / **AWS Secrets Manager** / **Doppler**.

---

## Blue-green and zero-downtime

A simple recipe with Docker + nginx:

1. Run the new image on a different port (the "green" deployment)
2. Hit `/health` until it's green
3. Switch nginx upstream to the new port (atomic config reload)
4. Stop the old container

Higher-level platforms (Fly, ECS, Kubernetes, Cloud Run) do this for you with a few config lines.

---

## Fabric — when Ansible is overkill

For a one-server deploy, **Fabric** lets you run shell commands over SSH from a Python script:

```python
# fabfile.py
from fabric import task

@task
def deploy(c):
    with c.cd("/srv/app"):
        c.run("git pull")
        c.run("docker compose pull")
        c.run("docker compose up -d")
        c.run("docker image prune -f")
```

```bash
uv run fab -H deploy@example.com deploy
```

Less ceremony than Ansible, more structure than shell scripts. Good for a small toolbox of admin commands.

---

## What about Heroku-style platforms?

Managed platforms ("PaaS") collapse most of this:

| Platform | Push to deploy via |
| --- | --- |
| **Fly.io** | `flyctl deploy` (uses your Dockerfile) |
| **Render** | git push triggers build |
| **Railway** | git push triggers build |
| **Cloud Run** | `gcloud run deploy` (with a container) |
| **App Runner** (AWS) | git or registry trigger |

Pros: zero-ops for small apps. Cons: less control, vendor lock-in, costs scale up.

For a side project or MVP, **start managed**. Move down the stack only when the bill or limits push you.

---

## Observability — minimum

- **Logs** to stdout/stderr — let the orchestrator collect them
- **Errors** to **Sentry** (or similar)
- **Metrics** — at least request count, p95 latency, 5xx rate
- **Health endpoint** — what the load balancer polls
- **Tracing** — OpenTelemetry, when the call graph gets non-trivial

You can't fix what you can't see. Ship logging on day one.

---

## A pragmatic deploy checklist

- [ ] CI builds and tests on every push
- [ ] Image tagged with both `latest` and the commit SHA
- [ ] Migrations run **before** swapping app containers
- [ ] Health endpoint exists and is checked post-deploy
- [ ] Secrets via env vars, never in the image
- [ ] Logs go to stdout
- [ ] Errors flow to Sentry / alerting
- [ ] Rollback is possible in one command (or automatic)
- [ ] Deploy frequency is high enough to keep PRs small

Tick the boxes that fit your scale. Don't tick all of them on day one.

---

## What's next

- **Data & AI** — pandas, scikit-learn (an aside on Python's other use cases)
- **LLM agents** — building with Anthropic / OpenAI APIs
