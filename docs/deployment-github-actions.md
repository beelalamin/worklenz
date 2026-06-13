# Deploying Worklenz with GitHub Actions

This guide takes you end‑to‑end: from generating keys, through configuring
GitHub secrets, to a fully automated deploy that fires on every push to `main`.

The pipeline is already defined in [`.github/workflows/deploy.yml`](../.github/workflows/deploy.yml).
It does three things on every push to `main` (or a manual trigger):

1. Builds the **backend** and **frontend** Docker images.
2. Pushes them to **Docker Hub**.
3. SSHes into your server, pulls the new images, and restarts the stack.

```
┌──────────────┐   push to main    ┌─────────────────────┐
│  GitHub repo │ ────────────────▶ │  GitHub Actions      │
└──────────────┘                   │  build-and-push job  │
                                   └──────────┬───────────┘
                                              │ docker push
                                              ▼
                                   ┌──────────────────────┐
                                   │      Docker Hub       │
                                   └──────────┬───────────┘
                                              │ docker compose pull
                                   ┌──────────▼───────────┐
                          SSH ───▶ │   Your server (root)  │
                                   │  docker compose up -d │
                                   └───────────────────────┘
```

---

## Part 1 — Prerequisites

- A server you can reach over SSH **as root** (you said you have this).
- A [Docker Hub](https://hub.docker.com) account.
- Admin access to this GitHub repository (to add secrets).
- A domain name pointing at the server (recommended, required for SSL).

> **Important — Docker Hub username must match the compose image names.**
> `docker-compose.yaml` references `beelalamin/worklenz-backend` and
> `beelalamin/worklenz-frontend`. The deploy workflow pushes to
> `<DOCKERHUB_USERNAME>/worklenz-backend` and `<…>/worklenz-frontend`.
> These must line up. Either:
> - Use the Docker Hub account `beelalamin`, **or**
> - Change the `image:` lines in `docker-compose.yaml` to your own namespace
>   and commit that change.

---

## Part 2 — One‑time server setup (run as root)

SSH into the server and prepare it once. After this, every deploy is automatic.

### 2.1 Install Docker + Compose plugin

```bash
# Docker Engine + Compose plugin (Debian/Ubuntu)
curl -fsSL https://get.docker.com | sh
docker compose version   # verify the compose plugin is present
```

### 2.2 Clone the repository

Pick a stable path — this becomes your `DEPLOY_PATH` secret later.

```bash
mkdir -p /opt
git clone https://github.com/Worklenz/worklenz.git /opt/worklenz
cd /opt/worklenz
```

### 2.3 Create and fill in `.env`

```bash
cp .env.example .env
```

Generate strong secrets and paste them into `.env`:

```bash
# Run 3 times for SESSION_SECRET, COOKIE_SECRET, JWT_SECRET
openssl rand -hex 32
# Strong DB / Redis / MinIO passwords
openssl rand -base64 24
```

Edit `.env` and set **at minimum**:

| Variable | Set to |
|---|---|
| `DOMAIN` | your domain, e.g. `app.example.com` |
| `VITE_API_URL` | `https://app.example.com` |
| `VITE_SOCKET_URL` | `wss://app.example.com` |
| `FRONTEND_URL` | `https://app.example.com` |
| `SOCKET_IO_CORS` | `https://app.example.com` |
| `S3_PUBLIC_URL` | `https://app.example.com` |
| `DB_PASSWORD` | a strong password |
| `REDIS_PASSWORD` | a strong password |
| `SESSION_SECRET` / `COOKIE_SECRET` / `JWT_SECRET` | the `openssl rand -hex 32` values |
| `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` | MinIO root creds (a username + strong password) |

> **Enable the bundled services.** The `redis` and `minio` containers live
> behind the `express` Compose profile, and `certbot` behind `ssl`. The deploy
> script runs `docker compose up -d` without `--profile` flags, so you must
> opt in via `.env`. Add this line so those services start automatically:
>
> ```bash
> COMPOSE_PROFILES=express        # add ,ssl once you set up HTTPS
> ```

### 2.4 First boot (manual, to confirm it works)

```bash
cd /opt/worklenz
docker compose up -d
docker compose ps          # everything should become healthy
curl -f http://localhost/health
```

Once this works manually, GitHub Actions can take over.

---

## Part 3 — Keys & tokens

### 3.1 Docker Hub access token

1. Docker Hub → **Account Settings → Security → New Access Token**.
2. Description: `worklenz-ci`, permissions: **Read & Write**.
3. Copy the token — you only see it once. This becomes `DOCKERHUB_TOKEN`.

### 3.2 SSH key pair for GitHub Actions

Generate a **dedicated** key for CI (don't reuse your personal key). Do this on
your local machine:

```bash
ssh-keygen -t ed25519 -C "github-actions-worklenz" -f ~/.ssh/worklenz_deploy -N ""
```

This creates:
- `~/.ssh/worklenz_deploy`     → **private** key → goes into `SSH_KEY` secret
- `~/.ssh/worklenz_deploy.pub` → **public** key → goes onto the server

Authorize the public key on the server (as root):

```bash
ssh-copy-id -i ~/.ssh/worklenz_deploy.pub root@YOUR_SERVER_IP
# or manually:
cat ~/.ssh/worklenz_deploy.pub | ssh root@YOUR_SERVER_IP \
  'mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys'
```

Test it:

```bash
ssh -i ~/.ssh/worklenz_deploy root@YOUR_SERVER_IP 'docker compose version'
```

> The workflow uses `appleboy/ssh-action`. Provide the **full private key**
> including the `-----BEGIN ... PRIVATE KEY-----` / `-----END ...-----` lines.

---

## Part 4 — Configure GitHub repository secrets

Go to **GitHub repo → Settings → Secrets and variables → Actions → New repository secret**
and add each of the following.

### Required secrets

| Secret | Value | Example |
|---|---|---|
| `DOCKERHUB_USERNAME` | Docker Hub username (must match compose image prefix) | `beelalamin` |
| `DOCKERHUB_TOKEN` | Docker Hub access token from Part 3.1 | `dckr_pat_…` |
| `SSH_HOST` | Server IP or hostname | `203.0.113.10` |
| `SSH_USER` | SSH user (you're using root) | `root` |
| `SSH_KEY` | **Private** key from Part 3.2 (entire file contents) | `-----BEGIN OPENSSH PRIVATE KEY-----…` |
| `DEPLOY_PATH` | Absolute path to the cloned repo on the server | `/opt/worklenz` |

### Optional secret

| Secret | Value | Default if omitted |
|---|---|---|
| `SSH_PORT` | SSH port | `22` |

To paste the private key as a secret:

```bash
cat ~/.ssh/worklenz_deploy   # copy the whole output, including BEGIN/END lines
```

---

## Part 5 — How the deploy runs

### Triggers

- **Automatic:** every push to `main`.
- **Manual:** GitHub repo → **Actions → Deploy → Run workflow**. You can
  optionally enter a git ref (branch, tag, or SHA) to build and deploy.

### What happens on the server

The SSH step runs (from `deploy.yml`):

```bash
cd "$DEPLOY_PATH"
git fetch --all --prune
git checkout main
git pull --ff-only origin main          # sync compose / nginx / sql changes
docker compose pull backend frontend    # pull the freshly built images
docker compose up -d --remove-orphans   # recreate only what changed
# waits up to ~150s for worklenz-backend healthcheck to report "healthy"
docker image prune -f                    # reclaim disk from old layers
```

Images are tagged twice on every build — `:latest` and `:<git-sha>` — so you
always have an immutable tag to roll back to.

> **Concurrency:** the workflow uses a `deploy-production` concurrency group
> with `cancel-in-progress: false`, so two deploys never run at once — a new
> one queues behind the running one instead of interrupting it.

### Monitoring a deploy

- Watch it live: GitHub repo → **Actions** → click the running run.
- On the server:
  ```bash
  cd /opt/worklenz
  docker compose ps
  docker compose logs -f backend
  ```

---

## Part 6 — Enable HTTPS (recommended)

1. Point your domain's A record at the server IP.
2. On the server, add `ssl` to the profiles and issue a certificate:
   ```bash
   cd /opt/worklenz
   # in .env:  COMPOSE_PROFILES=express,ssl
   docker compose run --rm certbot certonly \
     --webroot -w /var/www/certbot \
     -d app.example.com --email you@example.com --agree-tos --no-eff-email
   docker compose up -d
   ```
3. Confirm the nginx config under `nginx/conf.d/` references the issued cert
   (see [`DOCKER_SETUP.md`](../DOCKER_SETUP.md) for the SSL details). Certbot
   auto‑renews every 12h via the `certbot` container.

---

## Part 7 — Rollback

To roll back to a previous build, on the server pin the image tags to a known
good git SHA and restart:

```bash
cd /opt/worklenz
export BACKEND_VERSION=<good-sha>
export FRONTEND_VERSION=<good-sha>
docker compose up -d backend frontend
```

(`BACKEND_VERSION` / `FRONTEND_VERSION` are honored by `docker-compose.yaml`.)
To make it permanent, set those values in `.env`.

---

## Part 8 — Troubleshooting

| Symptom | Likely cause / fix |
|---|---|
| `denied: requested access to the resource is denied` during push | `DOCKERHUB_USERNAME`/`DOCKERHUB_TOKEN` wrong, or token lacks write scope. |
| Server pulls an image that doesn't exist | `DOCKERHUB_USERNAME` doesn't match the `image:` prefix in `docker-compose.yaml`. |
| `Permission denied (publickey)` in the deploy step | `SSH_KEY` is wrong/incomplete, or the **public** key isn't in the server's `~/.ssh/authorized_keys`. |
| `redis`/`minio` containers not running | `COMPOSE_PROFILES=express` missing from `.env`. |
| "Backend did not become healthy in time" | Check `docker compose logs backend`; usually a bad `.env` value (DB password, secrets) or a failed migration. |
| Storage/attachments break | Ensure `S3_*` and `AWS_*` values match and `S3_PUBLIC_URL` is your public origin (see compose comments). |

---

## Secrets checklist (TL;DR)

```
DOCKERHUB_USERNAME   ✅ matches compose image prefix
DOCKERHUB_TOKEN      ✅ Read & Write
SSH_HOST             ✅ server IP/hostname
SSH_USER             ✅ root
SSH_KEY              ✅ full private key (BEGIN…END)
DEPLOY_PATH          ✅ /opt/worklenz
SSH_PORT             ⬜ optional (defaults to 22)
```

Once these are set and the server is prepped, push to `main` and watch the
**Actions** tab — the rest is automatic.
