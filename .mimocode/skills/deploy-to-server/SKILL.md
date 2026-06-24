---
description: Deploy email-otp-autofill agent to the Tencent Cloud CVM server via rsync + SSH + docker compose. Handles code sync, rebuild, restart, and health verification.
name: deploy-to-server
---

# Deploy to Server

Deploy the agent service to the remote Tencent Cloud CVM server. The server runs the agent in Docker via `docker compose`.

## Server details

- **Host**: `tencent-cvm` (SSH alias, see `~/.ssh/config`)
- **Deploy path**: `/opt/email-otp-autofill`
- **Docker compose**: `docker-compose.yml` at deploy path
- **Public URL**: `https://otp.razet.me` (via Cloudflare tunnel)

## When to use

- After pushing agent code changes to `main`
- When user says "deploy", "部署", "发布", "更新服务器"
- When GitHub Actions deploy fails and manual deploy is needed

## Steps

### 1. Sync code to server

```bash
rsync -az --delete \
  --exclude 'node_modules' --exclude 'dist' --exclude '.git' \
  -e ssh \
  ./agent ./docker-compose.yml ./.env.example \
  tencent-cvm:/opt/email-otp-autofill/
```

Only sync `agent/`, `docker-compose.yml`, and `.env.example`. Do NOT sync `.env` (secrets stay on server).

### 2. Rebuild and restart on server

```bash
ssh tencent-cvm 'cd /opt/email-otp-autofill && \
  docker compose build agent 2>&1 | tail -15 && \
  docker compose up -d agent 2>&1 | tail -5'
```

### 3. Verify deployment

```bash
# Check container is running
ssh tencent-cvm 'cd /opt/email-otp-autofill && docker compose ps'

# Check health endpoint (public)
curl -s -o /dev/null -w "http_code=%{http_code}\n" https://otp.razet.me/v1/status \
  -H "x-otp-agent-client: email-otp-autofill"

# Check container logs for errors
ssh tencent-cvm 'cd /opt/email-otp-autofill && docker compose logs agent 2>&1 | tail -20'
```

### 4. Verify code version matches local

```bash
echo "local:  $(git -C . rev-parse --short HEAD)"
echo "server: $(ssh tencent-cvm 'cd /opt/email-otp-autofill && git rev-parse --short HEAD')"
```

Both should match. If server is behind, the GitHub Actions CI may not have triggered, or rsync didn't include a git repo (common — rsync doesn't copy `.git`).

## Troubleshooting

- **Build fails**: Check `docker compose build agent` output for missing deps or compilation errors.
- **Container won't start**: `ssh tencent-cvm 'cd /opt/email-otp-autofill && docker compose logs agent'`
- **Port conflict**: `ssh tencent-cvm 'docker compose ps'` to see if another container uses the port.
- **Server `.env` out of sync**: SSH in and check `cat /opt/email-otp-autofill/.env` (values are secrets — don't print full values, just confirm keys exist).

## Important notes

- Server `.env` contains secrets (`OTP_ADMIN_TOKEN`, `OTP_AGENT_MASTER_KEY`, etc.) — never rsync `.env` from local.
- The server has its own git repo initialized from rsync'd files. If the server doesn't have `.git`, it's normal (rsync excludes `.git` by default).
- If GitHub Actions deploy (`deploy-agent.yml`) is working, manual deploy is usually unnecessary.
