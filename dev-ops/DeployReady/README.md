# Kora Analytics API — DevOps Implementation

A Node.js REST API containerised with Docker, deployed to AWS EC2, and delivered via an automated GitHub Actions CI/CD pipeline.

**Live endpoint:** `http://13.60.70.231/health`

---

## Architecture Overview

```
Developer pushes to main
         │
         ▼
┌─────────────────────────────────────────┐
│          GitHub Actions Pipeline        │
│                                         │
│  [Test] → [Build] → [Push] → [Deploy]   │
└─────────────────────────────────────────┘
         │
         ▼ SSH
┌─────────────────────────────────────────┐
│          AWS EC2 (t3.micro)             │
│          Amazon Linux 2023              │
│          IP: 13.60.70.231               │
│                                         │
│  Docker container: kora-api             │
│  Host port 80 → container port 3000     │
└─────────────────────────────────────────┘
         │
         ▼
  Image stored at:
  ghcr.io/engkarangwa/kora-analytics-api
```

---

## API Endpoints

| Method | Route      | Description                            |
|--------|------------|----------------------------------------|
| GET    | `/health`  | Returns `{ "status": "ok" }`           |
| GET    | `/metrics` | Returns uptime and memory usage        |
| POST   | `/data`    | Accepts a JSON body and echoes it back |

---

## Part 1 — Containerisation

### Run locally

```bash
cd dev-ops/DeployReady
cp .env.example .env
docker compose up --build
```

Test the endpoints:
```bash
curl http://localhost:3000/health
curl http://localhost:3000/metrics
```

### Key decisions

**Dockerfile:**
- `node:20-alpine` as base image — Alpine keeps the image small and reduces attack surface
- Dependencies installed with `npm ci` for reproducible builds from the lockfile
- Non-root user `nodeuser` created and assigned file ownership via `--chown` on `COPY`
- `HEALTHCHECK` uses `wget` (built into Alpine) to probe `/health` every 30 seconds
- `USER nodeuser` set before `CMD` — the process never runs as root

**docker-compose.yml:**
- Loads variables from `.env` file at runtime
- `restart: unless-stopped` — auto-recovers from crashes, stays stopped after a manual stop
- Healthcheck mirrors the Dockerfile so Compose also tracks container health

---

## Part 2 — CI/CD Pipeline

Pipeline file: [`.github/workflows/deploy.yml`](../../.github/workflows/deploy.yml)

Triggers on every push to `main` that changes files inside `dev-ops/DeployReady/`.

### Pipeline stages

| Stage | What it does | Fails if |
|-------|-------------|----------|
| **Test** | Runs `npm test` (Jest) | Any test fails |
| **Build** | Builds Docker image, tags with commit SHA + `latest` | Build error |
| **Push** | Pushes both tags to `ghcr.io` | Auth or network error |
| **Deploy** | SSHs into EC2, pulls new image, restarts container, checks `/health` | Health check fails after boot |

### Secrets required

| Secret | Purpose |
|---|---|
| `EC2_HOST` | EC2 public IP (`13.60.70.231`) |
| `EC2_USER` | SSH username (`ec2-user`) |
| `EC2_SSH_KEY` | Private key for SSH access |
| `GHCR_USERNAME` | GitHub username (`EngKARANGWA`) |
| `GHCR_TOKEN` | GitHub PAT with `read:packages` scope |

No secrets are stored in code or committed files.

---

## Part 3 — AWS Deployment

Full setup details in [DEPLOYMENT.md](./DEPLOYMENT.md).

**Cloud provider:** AWS — chosen because of free tier availability, mature EC2 tooling, and native integration with GitHub Actions via SSH.

### Infrastructure

- **Instance:** EC2 `t3.micro`, Amazon Linux 2023 — chosen over `t2.micro` because t3 is the current-generation burstable instance with better baseline CPU performance and lower cost per vCPU
- **Public IP:** `13.60.70.231`
- **Container registry:** GitHub Container Registry (`ghcr.io`)

### Security group rules

| Type | Port | Source                   |
|------|------|--------------------------|
| HTTP | 80   | `0.0.0.0/0`              |
| SSH  | 22   | `154.68.64.230/32` (owner IP only) |

### Verify the live deployment

```bash
curl http://13.60.70.231/health
# {"status":"ok"}
```

---

## Project Structure

```
dev-ops/DeployReady/
├── app/
│   ├── index.js          # Express API (not modified)
│   ├── index.test.js     # Jest tests
│   └── package.json
├── Dockerfile            # Non-root user, healthcheck, alpine base
├── docker-compose.yml    # Local development setup
├── .env.example          # Environment variable template
├── DEPLOYMENT.md         # AWS setup and operations guide
└── README.md             # This file

.github/
└── workflows/
    └── deploy.yml        # 4-stage CI/CD pipeline
```

---

## Pre-Submission Checklist

- [x] `docker compose up --build` starts the app locally
- [x] `.env.example` is committed — real `.env` is gitignored
- [x] At least one successful pipeline run visible in GitHub Actions tab
- [x] `GET http://13.60.70.231/health` returns `{ "status": "ok" }`
- [x] No secrets or `.pem` files committed to the repository
- [x] SSH port 22 is **not** open to `0.0.0.0/0`
- [x] `DEPLOYMENT.md` covers all four required points
- [x] This README replaces the original assignment README
- [x] Commit history shows incremental progress
