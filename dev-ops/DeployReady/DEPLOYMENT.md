# Deployment Guide — Kora Analytics API

## Architecture Overview

```
GitHub (main branch)
       │
       ▼
GitHub Actions Pipeline
  ├── 1. npm test
  ├── 2. docker build + tag with commit SHA
  ├── 3. docker push → ghcr.io
  └── 4. SSH → EC2 → docker pull + restart
                         │
                         ▼
                  EC2 t2.micro (Amazon Linux 2023)
                  Docker container: kora-api
                  Port 80 → 3000
```

---

## 1. EC2 Instance Setup

**Instance details:**
- AMI: Amazon Linux 2023
- Instance type: t2.micro (free tier)
- Region: us-east-1 (or your chosen region)
- Key pair: downloaded `.pem` file stored securely (never committed)

**Security Group — `kora-api-sg`:**

| Type | Protocol | Port | Source          | Reason                        |
|------|----------|------|-----------------|-------------------------------|
| HTTP | TCP      | 80   | 0.0.0.0/0       | Public API access             |
| SSH  | TCP      | 22   | Your IP only    | Admin access — not open world |

SSH port 22 is restricted to a single IP. It is **not** open to `0.0.0.0/0`.

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
echo "<GHCR_TOKEN>" | docker login ghcr.io -u <GITHUB_USERNAME> --password-stdin
docker pull ghcr.io/<github-username>/kora-analytics-api:latest
```

Start the container:
```bash
docker run -d \
  --name kora-api \
  --restart unless-stopped \
  -p 80:3000 \
  -e PORT=3000 \
  -e NODE_ENV=production \
  ghcr.io/<github-username>/kora-analytics-api:latest
```

After each deploy the pipeline handles the pull and restart automatically via SSH.

---

## 3. Checking if the Container is Running

```bash
# See container status and health
docker ps

# Check health status specifically
docker inspect --format='{{.State.Health.Status}}' kora-api

# Confirm the API responds
curl http://localhost/health
```

Expected output from the health check:
```json
{"status":"ok"}
```

From outside the server (replace with your EC2 public IP):
```bash
curl http://<EC2_PUBLIC_IP>/health
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

The following secrets must be set in GitHub → Settings → Secrets → Actions:

| Secret          | Description                                      |
|-----------------|--------------------------------------------------|
| `EC2_HOST`      | EC2 public IP address                            |
| `EC2_USER`      | SSH username (`ec2-user` on Amazon Linux)        |
| `EC2_SSH_KEY`   | Contents of the `.pem` private key file          |
| `GHCR_USERNAME` | GitHub username for GHCR authentication          |
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
