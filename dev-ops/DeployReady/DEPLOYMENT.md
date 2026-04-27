# Deployment Guide — Kora Analytics API

## Architecture Overview

```
GitHub (main branch)
       │
       ▼
GitHub Actions Pipeline
  ├── 1. npm test
  ├── 2. docker build + tag with commit SHA
  ├── 3. docker push → ghcr.io/engkarangwa/kora-analytics-api
  └── 4. SSH → EC2 → docker pull + restart
                         │
                         ▼
                  EC2 t3.micro (Amazon Linux 2023)
                  Public IP: 13.60.70.231
                  Docker container: kora-api
                  Port 80 → 3000
```

---

## 1. EC2 Instance Setup

**Instance details:**
- AMI: Amazon Linux 2023
- Instance type: t3.micro — current-generation burstable instance, chosen over t2.micro because t3 offers better baseline CPU performance, uses the newer Nitro hypervisor, and is eligible for the AWS free tier
- Public IP: 13.60.70.231
- Key pair: `kora.pem` — stored securely, never committed to the repository

**Security Group — `kora-api-sg`:**

| Type | Protocol | Port | Source           | Reason                        |
|------|----------|------|------------------|-------------------------------|
| HTTP | TCP      | 80   | 0.0.0.0/0        | Public API access             |
| SSH  | TCP      | 22   | 154.68.64.230/32 | Admin access — restricted to owner IP only |

SSH port 22 is restricted to a single IP (`154.68.64.230/32`). It is **not** open to `0.0.0.0/0`.

---

## 2. Installing Docker and Pulling the Image

SSH into the instance:
```bash
ssh -i kora.pem ec2-user@13.60.70.231
```

Install and start Docker:
```bash
sudo dnf update -y
sudo dnf install -y docker
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker ec2-user
```

Log out and back in, then authenticate with GHCR and pull the image:
```bash
echo "<GHCR_TOKEN>" | docker login ghcr.io -u EngKARANGWA --password-stdin
docker pull ghcr.io/engkarangwa/kora-analytics-api:latest
```

Start the container:
```bash
docker run -d \
  --name kora-api \
  --restart unless-stopped \
  -p 80:3000 \
  -e PORT=3000 \
  -e NODE_ENV=production \
  ghcr.io/engkarangwa/kora-analytics-api:latest
```

After each deploy the pipeline handles the pull and restart automatically via SSH.

---

## 3. Checking if the Container is Running

```bash
# See container status and health
docker ps

# Check health status specifically
docker inspect --format='{{.State.Health.Status}}' kora-api

# Confirm the API responds locally on the server
curl http://localhost:3000/health
```

Expected output from the health check:
```json
{"status":"ok"}
```

From outside the server:
```bash
curl http://13.60.70.231/health
```

---

## 4. Viewing Application Logs

```bash
# Stream live logs
docker logs -f kora-api

# Show last 50 lines
docker logs --tail 50 kora-api

# Show logs with timestamps
docker logs -t kora-api
```

---

## Pipeline Secrets Reference

The following secrets are set in GitHub → Settings → Secrets → Actions:

| Secret          | Description                                      |
|-----------------|--------------------------------------------------|
| `EC2_HOST`      | `13.60.70.231`                                   |
| `EC2_USER`      | `ec2-user`                                       |
| `EC2_SSH_KEY`   | Contents of the `kora.pem` private key file      |
| `GHCR_USERNAME` | `EngKARANGWA`                                    |
| `GHCR_TOKEN`    | GitHub PAT with `read:packages` scope            |

No secrets are stored in the repository or any committed file.

---

## Useful Commands

```bash
# Restart the container manually
docker restart kora-api

# Stop the container
docker stop kora-api

# Remove the container
docker rm kora-api

# List all images on the server
docker images

# Free up disk space (remove unused images)
docker image prune -f
```
