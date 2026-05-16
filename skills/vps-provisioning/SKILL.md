---
name: vps-provisioning
description: "Use when provisioning a fresh VPS, installing server software, or setting up an AI-agent host. Covers Ubuntu/Debian: SSH, Docker, Nginx, Node.js, Python, PostgreSQL, Redis, SSL, monitoring, and security."
license: MIT
compatibility: Requires an SSH client and a fresh Ubuntu/Debian VPS with root access
metadata:
  author: Murugadoss
  version: "1.1.0"
  tags: "vps devops provisioning server docker nginx security deployment"
  category: DevOps
---

# VPS Provisioning & Installation

An AI-agent-readable guide for provisioning a fresh VPS from scratch. Every recipe is a self-contained workflow an agent can execute step-by-step.

## When to Use

- You have a **fresh Ubuntu/Debian VPS** that needs bootstrapping (Docker, Nginx, Node.js, etc.)
- You need to **reprovision** a server to a known state after a reset
- You're setting up a **host for AI agents** (OpenClaw, Hermes gateway, MCP servers)
- You need to **install a specific stack** (LAMP, JAMstack, ML inference server)

**Don't use for:** Windows Server, Alpine Linux (packages differ significantly), or already-provisioned servers where you just need one package (use `apt` directly).

## Prerequisites

| Requirement | Check |
|---|---|
| VPS IP address or hostname | `ping <vps-ip>` responds |
| Root password **or** SSH key deployed | Ask provider for root access |
| SSH client installed | `which ssh` |
| DNS (optional) | A record pointing to VPS IP |
| This session uses `pty=true` | For interactive SSH prompts |

## Quick Start: Full Bootstrap (15 min)

Run these in order on a fresh VPS. Each section is idempotent — safe to re-run.

### 0. Distro Detection — Check Before You Start

```bash
# Run on VPS immediately after first SSH login
cat /etc/os-release | grep -E '^ID=|^VERSION_ID='
```

Write down the values — several commands in this skill differ between **Ubuntu 24.04+** and older releases (most notably the SSH service name). If `VERSION_ID` is 24.04 or higher, make a mental note: `ssh` not `sshd`.

### 1. Initial SSH Access & User Setup

```bash
# Connect as root (you'll be prompted for password)
ssh root@<VPS-IP>

# Create a deploy user with sudo
adduser deploy
usermod -aG sudo deploy

# Copy your local SSH key to the deploy user
# Run on YOUR machine (not the VPS):
ssh-copy-id deploy@<VPS-IP>

# Now reconnect as deploy
ssh deploy@<VPS-IP>
```

**Pitfall:** If `ssh-copy-id` isn't available, manually:
```bash
# On your machine:
cat ~/.ssh/id_ed25519.pub | ssh deploy@<VPS-IP> "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

### 2. System Update & Hardening

```bash
# Run as root or with sudo
sudo apt update && sudo apt upgrade -y
sudo apt install -y ufw fail2ban unattended-upgrades curl wget git htop net-tools

# Configure firewall
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh
sudo ufw allow http
sudo ufw allow https
sudo ufw --force enable

# Harden SSH via drop-in (wins over distro defaults; survives package upgrades)
sudo install -d -m 0755 /etc/ssh/sshd_config.d
# printf (single line) instead of heredoc — heredocs break when an agent wraps
# the whole block in `ssh user@host "..."` because the outer quotes close early.
printf '%s\n' \
  'PermitRootLogin prohibit-password' \
  'PasswordAuthentication no' \
  'KbdInteractiveAuthentication no' \
  | sudo tee /etc/ssh/sshd_config.d/10-hardening.conf > /dev/null

# Validate config BEFORE restart — broken config + restart can lock you out
sudo sshd -t || { echo "ABORT: sshd config invalid — fix before restarting"; exit 1; }

# Restart SSH — service name differs by distro version
# Ubuntu 24.04+ uses `ssh`, older uses `sshd`
if systemctl list-unit-files | grep -q '^ssh\.service'; then
  sudo systemctl restart ssh
elif systemctl list-unit-files | grep -q '^sshd\.service'; then
  sudo systemctl restart sshd
else
  echo "WARNING: Could not find ssh or sshd service"
fi

# Enable auto security updates
sudo dpkg-reconfigure --priority=low unattended-upgrades
```

**Danger zone:** Disabling password auth locks you out if your SSH key isn't deployed. **BEFORE running the hardening commands above**, open a SECOND SSH session as `deploy` and confirm it works without a password (`ssh -o PreferredAuthentications=publickey deploy@<VPS-IP>`). Keep that session open until you've confirmed the new config works in a third session. `sshd -t` (above) catches syntax errors but not policy mistakes.

### 3. Swap (Low-RAM VPS)

```bash
# 2GB swap — adjust size as needed. Idempotent: skips if /swapfile already present.
if [ ! -f /swapfile ]; then
  sudo fallocate -l 2G /swapfile
  sudo chmod 600 /swapfile
  sudo mkswap /swapfile
  sudo swapon /swapfile
fi
grep -qxF '/swapfile none swap sw 0 0' /etc/fstab || \
  echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
# Set swappiness to 10 (default 60, lower = less swapping)
grep -qxF 'vm.swappiness=10' /etc/sysctl.conf || \
  echo 'vm.swappiness=10' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

### 4. Docker & Docker Compose

```bash
# Install Docker (official method)
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Add deploy user to docker group (avoids sudo each time).
# Hardcoded to `deploy` — when this section runs under sudo, $USER would be root.
sudo usermod -aG docker deploy

# Install Docker Compose plugin
sudo apt install -y docker-compose-plugin

# Test
docker --version
docker compose version
docker run hello-world
```

**Pitfall:** The group change only takes effect in a new login shell. After `sudo usermod -aG docker`, log out and back in: `exit` then `ssh deploy@<VPS-IP>`. Don't test `docker ps` before reconnecting — it will fail and confuse you.

### 5. Nginx + Certbot (SSL)

```bash
sudo apt install -y nginx certbot python3-certbot-nginx

# Start and enable
sudo systemctl enable nginx
sudo systemctl start nginx

# Verify
curl localhost | grep nginx  # Should show "Welcome to nginx"

# Get SSL (requires DNS pointing here)
# sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com
```

### 6. Node.js (via nvm)

```bash
# Install nvm
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash

# Reload shell config
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"

# Install latest LTS
nvm install --lts
nvm use --lts
node --version  # Should be 22.x or higher
npm --version

# Optional: pnpm for faster installs
npm install -g pnpm
```

### 7. Python (pyenv + uv)

```bash
# Install build dependencies
sudo apt install -y build-essential libssl-dev zlib1g-dev \
  libbz2-dev libreadline-dev libsqlite3-dev libncursesw5-dev \
  xz-utils tk-dev libxml2-dev libxmlsec1-dev libffi-dev liblzma-dev

# Install pyenv
curl https://pyenv.run | bash

# Add to shell — guarded so re-runs don't duplicate entries
for line in \
  'export PYENV_ROOT="$HOME/.pyenv"' \
  'command -v pyenv >/dev/null || export PATH="$PYENV_ROOT/bin:$PATH"' \
  'eval "$(pyenv init -)"'; do
  grep -qxF "$line" ~/.bashrc || echo "$line" >> ~/.bashrc
done
# shellcheck disable=SC1090
source ~/.bashrc

# Install Python 3.12 (latest stable)
pyenv install 3.12
pyenv global 3.12

# Install uv (fast Python package manager)
curl -LsSf https://astral.sh/uv/install.sh | sh
source ~/.bashrc
uv --version
```

### 8. PostgreSQL

```bash
sudo apt install -y postgresql postgresql-contrib

# Start and enable
sudo systemctl enable postgresql
sudo systemctl start postgresql

# Create a deploy database user
sudo -u postgres createuser --interactive
# Answer: n (not superuser), n (no create db), n (no create roles)

# Or non-interactively. Generate a strong password and SAVE IT (printed to stdout).
# Do not commit this password anywhere; store it in your secret manager.
PGPASS="$(openssl rand -base64 24)"
echo "Postgres deploy user password: $PGPASS"
sudo -u postgres psql -c "CREATE USER deploy WITH PASSWORD '$PGPASS';"
sudo -u postgres psql -c "CREATE DATABASE myapp OWNER deploy;"
unset PGPASS

# Test
sudo -u postgres psql -c "\l"
```

### 9. Redis

```bash
sudo apt install -y redis-server

# Configure for systemd
sudo sed -i 's/^supervised no/supervised systemd/' /etc/redis/redis.conf

# Confirm Redis is bound to loopback only (default — verify, don't trust).
grep -E '^\s*bind\b' /etc/redis/redis.conf  # should show 127.0.0.1 ::1

# Set a password. Mandatory even on loopback-only: containers and local users
# on the box can still reach Redis without auth otherwise.
REDISPASS="$(openssl rand -base64 32)"
echo "Redis password: $REDISPASS"   # save this in your secret manager
# Replace any existing requirepass line OR append one if not present.
if grep -qE '^\s*#?\s*requirepass\b' /etc/redis/redis.conf; then
  sudo sed -i "s|^\s*#\?\s*requirepass.*|requirepass $REDISPASS|" /etc/redis/redis.conf
else
  echo "requirepass $REDISPASS" | sudo tee -a /etc/redis/redis.conf > /dev/null
fi
unset REDISPASS

sudo systemctl enable redis-server
sudo systemctl restart redis-server

# Test — should error without -a, then succeed with the password
redis-cli ping || echo "Expected: NOAUTH error. Now retry with: redis-cli -a <password> ping"
```

### 10. Monitoring Stack (Netdata)

```bash
# One-line Netdata install (HTTPS; script is from netdata.cloud, audit before running in CI)
bash <(curl -Ss https://my-netdata.io/kickstart.sh)
```

**Do NOT** `ufw allow 19999/tcp`. Netdata ships with no authentication and exposes
detailed host, process, and network metrics. Three safe access patterns, pick one:

```bash
# (a) SSH tunnel — preferred for occasional access. From your laptop:
#     ssh -L 19999:localhost:19999 deploy@<VPS-IP>
#     then open http://localhost:19999

# (b) Restrict UFW to a specific source IP:
sudo ufw allow from <your-static-ip> to any port 19999 proto tcp

# (c) Nginx reverse proxy with basic auth + HTTPS (production):
sudo apt install -y apache2-utils
sudo htpasswd -c /etc/nginx/.netdata-htpasswd <username>
# then add a server{} block proxying / to http://127.0.0.1:19999 with auth_basic.
```

### 11. AI Agent Host Setup (OpenClaw / Hermes gateway)

```bash
# For OpenClaw (or any Python-based agent running as a service):
mkdir -p /home/deploy/agents
cd /home/deploy/agents

# Create a systemd service template
sudo tee /etc/systemd/system/agent.service << 'SERVICEEOF'
[Unit]
Description=AI Agent Service
After=network.target

[Service]
Type=simple
User=deploy
WorkingDirectory=/home/deploy/agents
Environment="PATH=/home/deploy/.local/bin:/usr/local/bin:/usr/bin:/bin"
ExecStart=/usr/bin/python3 -m <your-agent-module>
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
SERVICEEOF

sudo systemctl daemon-reload
```

## Recipes (Standalone)

### Recipe A: Deploy a Node.js App Behind Nginx

```bash
# 1. Build and test the app locally first
# 2. On the VPS:
git clone <your-repo> /home/deploy/app
cd /home/deploy/app
nvm use --lts
npm install
npm run build  # If SPA/frontend

# 3. PM2 for process management
npm install -g pm2
pm2 start npm --name "myapp" -- start
pm2 save
pm2 startup systemd  # Follow the instructions it prints

# 4. Nginx reverse proxy config
sudo tee /etc/nginx/sites-available/myapp << 'NGINXEOF'
server {
    listen 80;
    server_name yourdomain.com;

    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
    }
}
NGINXEOF

sudo ln -sf /etc/nginx/sites-available/myapp /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx

# 5. SSL
# sudo certbot --nginx -d yourdomain.com
```

### Recipe B: Docker Compose Stack

```bash
# On your machine, write a docker-compose.yml, then:
scp docker-compose.yml deploy@<VPS-IP>:/home/deploy/
ssh deploy@<VPS-IP>
cd /home/deploy
docker compose up -d
docker compose logs -f
```

Apply for: Databases, message queues, monitoring (Grafana/Prometheus), full-stack apps.

### Recipe C: Quick Fix — Server Running Out of Disk

```bash
# Check usage
df -h
du -sh /var/log/* | sort -rh | head -10

# Clean Docker
docker system prune -af  # Removes unused images, containers, networks

# Clean apt cache
sudo apt clean
sudo apt autoremove -y

# Rotate and compress logs
sudo logrotate -f /etc/logrotate.conf
sudo journalctl --vacuum-time=3d

# Check what's taking space
ncdu /  # Install with: sudo apt install -y ncdu
```

### Recipe D: Reset Server to Clean State

**DESTRUCTIVE — read the entire block before running any line.** This wipes
containers, volumes, and `/home/deploy` contents. Take a snapshot at your
provider first if any data matters. Every command below is commented out —
uncomment ONLY the ones you want, after confirming.

```bash
# Remove all Docker resources (containers, images, volumes — irreversible)
# docker system prune -a --volumes -f

# Remove all pm2 processes
# pm2 kill

# Remove app data but keep services. Note: keeps dotfiles; delete user instead
# if you want a fully clean state.
# rm -rf /home/deploy/app /home/deploy/agents

# Full uninstall of services (most destructive):
# sudo apt purge -y docker-ce nginx postgresql redis-server
```

## Troubleshooting Guide

| Problem | Likely Cause | Fix |
|---|---|---|
| `ssh: connect to host port 22: Connection refused` | VPS still booting | Wait 30s, retry. Check provider dashboard |
| `permission denied (publickey)` | SSH key not deployed | Reconnect as root with password, run `ssh-copy-id` |
| `sudo: unable to resolve host` | /etc/hosts entry wrong | `echo "127.0.1.1 $(hostname)" \| sudo tee -a /etc/hosts` |
| `docker: permission denied` | User not in docker group | Re-login shell: `exit` then SSH back in |
| Nginx `address already in use` | Apache/other on port 80 | `sudo lsof -i :80` then stop the other service |
| certbot `too many certificates` | Rate-limited | Certbot rate limit: 50/week. Use staging: `--staging` for testing |
| `No space left on device` | Docker/logs filling disk | Run Recipe C above |
| `Python 3.12 not found` | pyenv not in PATH | Source `~/.bashrc` or run `eval "$(pyenv init -)"` |

## Common Pitfalls

1. **SSH disconnection during long installs.** Use `screen` or `tmux` before starting: `sudo apt install -y tmux && tmux new -s provision`. If disconnected, reconnect and `tmux attach -t provision`.

2. **SSH service name varies by Ubuntu version.** Ubuntu 24.04+ uses `ssh.service`; older releases use `sshd.service`. If `sudo systemctl restart sshd` fails with `Unit sshd.service not found`, try `sudo systemctl restart ssh` instead. The skill's hardening section now includes a portable auto-detect, but if you're using a cloud-init script or one-liner, hardcode the right name for your target distro.

3. **Assuming `sudo` without password.** Some VPS images configure sudo to ask for password. Verify: `sudo -n whoami`. If it asks, wait for it — don't panic.

4. **Docker group doesn't take effect immediately.** The group membership changes apply to the next login shell. Run `newgrp docker` in the same session, or `exit` and re-SSH. Running `docker ps` immediately after `usermod` will fail.

5. **UFW blocks your SSH before you enable it.** The `sudo ufw allow ssh` rule (or `allow 22`) MUST come before `ufw --force enable`. Most guides get this right, but if you're editing rules remotely, always leave a second SSH session open as a safety net.

6. **Pyenv builds fail on minimal images.** Missing build deps (Section 7). Install them all before running `pyenv install`.

7. **Certbot DNS validation fails.** Make sure the A record points to your VPS IP and has propagated. Check with `dig yourdomain.com +short`.

8. **Container networking conflicts.** If Docker containers can't reach each other, check `docker network ls` and ensure they share a network: `docker network create mynet` and reference it in compose.

9. **Port already in use after reboot.** Services with `Restart=always` (Docker, PM2) restart and reclaim ports faster than Nginx. Set `After=network.target docker.service` in Nginx systemd override, or use `restart: unless-stopped` in compose.

10. **`curl … | bash` installers run unaudited code.** Five sections fetch and execute remote scripts (Docker, nvm, pyenv, uv, Netdata). All HTTPS, but the contents can change without notice. For agentic execution: pin versions where the project supports it (we pin nvm to v0.40.1), and in security-sensitive environments, mirror the script and audit before piping to a shell.

## Verification Checklist

After a full bootstrap, run through this:

- [ ] `ssh deploy@<VPS-IP>` — connects without password
- [ ] `sudo -n whoami` — returns `root` without prompting
- [ ] `ufw status verbose` — shows `22/tcp`, `80/tcp`, `443/tcp` ALLOW
- [ ] `docker run --rm hello-world` — Docker works without sudo
- [ ] `curl localhost | grep nginx` — Nginx welcome page
- [ ] `node --version` — ≥ 22.x
- [ ] `python3 --version` — ≥ 3.12
- [ ] `psql --version` — PostgreSQL client works
- [ ] `redis-cli ping` — returns PONG
- [ ] `pm2 list` — PM2 available (if installed)
- [ ] `systemctl status netdata | grep active` — Netdata running
- [ ] `df -h | grep /dev/sda` — < 60% disk usage
- [ ] `free -h | grep Swap` — swap active
- [ ] Browser: `http://<VPS-IP>` — Nginx landing page
- [ ] DNS (if set): `dig yourdomain.com +short` — resolves to VPS IP

## Compatibility Notes

- **Ubuntu 24.04+** ships the SSH service as `ssh.service`; older releases use `sshd.service`. The hardening section above auto-detects.
- **Debian 12 (bookworm)** generally works identically; package names match.
- **Not supported:** Alpine (apk, musl libc, OpenRC), CentOS/RHEL (dnf/yum), Windows Server.

---

*Built for the Agent Skills ecosystem. Ship fast, iterate in the open.*
