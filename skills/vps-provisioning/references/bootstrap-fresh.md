# Path B — Fresh-Host Bootstrap Recipes

Detailed install recipes for an **empty Ubuntu/Debian VPS**. Loaded only after [SKILL.md](../SKILL.md)'s Path B pre-flight confirms the host has no running containers or matching services.

If `docker ps` shows anything or a reverse proxy is already running, stop and use Path A in SKILL.md instead. Running the Docker section below on a host with a running Docker daemon will restart it and may disrupt running stacks.

## Prerequisites

| Requirement | Check |
|---|---|
| VPS IP or hostname | `ping <vps-ip>` responds |
| Root password OR SSH key already deployed | Ask provider for root access |
| SSH client on your local machine | `which ssh` |
| (Optional) DNS A record pointing to VPS IP | For SSL via certbot |

Use `tmux` or `screen` before long installs in case SSH drops:

```bash
sudo apt install -y tmux && tmux new -s provision
# If disconnected: ssh back in, then: tmux attach -t provision
```

## 0. Distro Detection

```bash
cat /etc/os-release | grep -E '^ID=|^VERSION_ID='
```

Note `VERSION_ID`. Ubuntu 24.04+ uses `ssh.service`; older uses `sshd.service`. The hardening commands below auto-detect.

## 1. Initial SSH Access & Deploy User

```bash
# From your local machine
ssh root@<VPS-IP>

# On the VPS, create a deploy user with sudo
adduser deploy
usermod -aG sudo deploy

# Back on your local machine, copy your SSH key:
ssh-copy-id deploy@<VPS-IP>

# If ssh-copy-id is unavailable, manual:
cat ~/.ssh/id_ed25519.pub | ssh deploy@<VPS-IP> \
  "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"

# Reconnect as deploy from this point forward
ssh deploy@<VPS-IP>
```

## 2. System Update & UFW + SSH Hardening

This is the same hardening from SKILL.md Path A, included here for the fresh-bootstrap flow. UFW rules here use 22/80/443 because a fresh host has no other services yet — **update them as you deploy each service in later sections.**

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y ufw fail2ban unattended-upgrades curl wget git htop net-tools

# Allow SSH FIRST (or you'll lock yourself out)
sudo ufw allow OpenSSH

# Default policies
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Enable. As you add services in later sections, run `sudo ufw allow <port>` for each.
sudo ufw --force enable

# SSH hardening via drop-in
sudo install -d -m 0755 /etc/ssh/sshd_config.d
printf '%s\n' \
  'PermitRootLogin prohibit-password' \
  'PasswordAuthentication no' \
  'KbdInteractiveAuthentication no' \
  | sudo tee /etc/ssh/sshd_config.d/10-hardening.conf > /dev/null

sudo sshd -t || { echo "ABORT: sshd config invalid"; exit 1; }

if systemctl list-unit-files | grep -q '^ssh\.service'; then
  sudo systemctl restart ssh
elif systemctl list-unit-files | grep -q '^sshd\.service'; then
  sudo systemctl restart sshd
fi

sudo dpkg-reconfigure --priority=low unattended-upgrades
sudo systemctl enable --now fail2ban
```

**Danger zone:** Verify key-based SSH works in a SECOND session BEFORE running the hardening drop-in. See SKILL.md A.2 for the full procedure.

## 3. Swap (Low-RAM VPS)

```bash
if [ ! -f /swapfile ]; then
  sudo fallocate -l 2G /swapfile
  sudo chmod 600 /swapfile
  sudo mkswap /swapfile
  sudo swapon /swapfile
fi
grep -qxF '/swapfile none swap sw 0 0' /etc/fstab || \
  echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab > /dev/null
grep -qxF 'vm.swappiness=10' /etc/sysctl.conf || \
  echo 'vm.swappiness=10' | sudo tee -a /etc/sysctl.conf > /dev/null
sudo sysctl -p
```

## 4. Docker & Docker Compose

```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Add deploy user to docker group (hardcoded — $USER is root under sudo).
sudo usermod -aG docker deploy

sudo apt install -y docker-compose-plugin

# Verify
docker --version
docker compose version
```

**Pitfall:** The docker group change takes effect only on a new login shell. `exit` and SSH back in before testing `docker ps`. Otherwise it'll fail with "permission denied" and confuse you.

**UFW reminder:** When you publish a container port with `-p 80:80`, Docker writes iptables rules ahead of UFW. The port is publicly reachable even if `ufw status` doesn't list it. To keep a Docker service private, bind to loopback: `-p 127.0.0.1:80:80`.

## 5. Nginx + Certbot (SSL)

```bash
sudo apt install -y nginx certbot python3-certbot-nginx
sudo systemctl enable --now nginx
sudo ufw allow 'Nginx Full'    # 80 + 443

# Verify
curl -s localhost | grep -i nginx

# SSL (requires DNS A record pointing here)
# sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com
```

## 6. Node.js (via nvm)

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash

export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"

nvm install --lts
nvm use --lts
node --version
npm --version

# Optional: faster installs
npm install -g pnpm
```

## 7. Python (pyenv + uv)

```bash
sudo apt install -y build-essential libssl-dev zlib1g-dev \
  libbz2-dev libreadline-dev libsqlite3-dev libncursesw5-dev \
  xz-utils tk-dev libxml2-dev libxmlsec1-dev libffi-dev liblzma-dev

curl https://pyenv.run | bash

# Append to ~/.bashrc only if not already present (idempotent)
for line in \
  'export PYENV_ROOT="$HOME/.pyenv"' \
  'command -v pyenv >/dev/null || export PATH="$PYENV_ROOT/bin:$PATH"' \
  'eval "$(pyenv init -)"'; do
  grep -qxF "$line" ~/.bashrc || echo "$line" >> ~/.bashrc
done
# shellcheck disable=SC1090
source ~/.bashrc

pyenv install 3.12
pyenv global 3.12

# Fast Python package manager
curl -LsSf https://astral.sh/uv/install.sh | sh
source ~/.bashrc
uv --version
```

## 8. PostgreSQL

```bash
sudo apt install -y postgresql postgresql-contrib
sudo systemctl enable --now postgresql

# Generate a strong password and SAVE IT (write to your secret manager).
PGPASS="$(openssl rand -base64 24)"
echo "Postgres deploy user password: $PGPASS"
sudo -u postgres psql -c "CREATE USER deploy WITH PASSWORD '$PGPASS';"
sudo -u postgres psql -c "CREATE DATABASE myapp OWNER deploy;"
unset PGPASS

# Verify
sudo -u postgres psql -c "\l"
```

**Postgres listens on 127.0.0.1:5432 by default.** Don't open 5432 in UFW unless you specifically need external access — connect over loopback or via app containers on the same Docker network.

## 9. Redis

```bash
sudo apt install -y redis-server
sudo sed -i 's/^supervised no/supervised systemd/' /etc/redis/redis.conf

# Verify loopback bind (default — confirm, don't assume)
grep -E '^\s*bind\b' /etc/redis/redis.conf   # expect: 127.0.0.1 ::1

# Mandatory password (loopback is not enough — local users and containers can reach it)
REDISPASS="$(openssl rand -base64 32)"
echo "Redis password: $REDISPASS"   # save this in your secret manager
if grep -qE '^\s*#?\s*requirepass\b' /etc/redis/redis.conf; then
  sudo sed -i "s|^\s*#\?\s*requirepass.*|requirepass $REDISPASS|" /etc/redis/redis.conf
else
  echo "requirepass $REDISPASS" | sudo tee -a /etc/redis/redis.conf > /dev/null
fi
unset REDISPASS

sudo systemctl enable redis-server
sudo systemctl restart redis-server

# Verify auth is enforced
redis-cli ping || echo "Expected NOAUTH; retry with: redis-cli -a <password> ping"
```

## 10. Monitoring (Netdata)

```bash
bash <(curl -Ss https://my-netdata.io/kickstart.sh)
```

**Do NOT `ufw allow 19999/tcp`.** Netdata ships with no auth and exposes detailed metrics. Pick one:

```bash
# (a) SSH tunnel — preferred for occasional access. From your laptop:
#     ssh -L 19999:localhost:19999 deploy@<VPS-IP>
#     then open http://localhost:19999

# (b) Restrict to a known source IP:
sudo ufw allow from <your-static-ip> to any port 19999 proto tcp

# (c) Nginx reverse proxy with basic auth + HTTPS:
sudo apt install -y apache2-utils
sudo htpasswd -c /etc/nginx/.netdata-htpasswd <username>
# then add a server{} block proxying / to http://127.0.0.1:19999 with auth_basic
```

## 11. AI Agent Host (OpenClaw / Hermes / MCP)

```bash
mkdir -p /home/deploy/agents

# systemd template — fill in ExecStart for your specific agent
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

For Docker-based agent stacks (e.g., OpenClaw): always set `--restart unless-stopped` on the container, AND make sure any process you launch *inside* the container (a gateway, a worker) is launched by the container's entrypoint or a supervisor (not by `docker exec` in a foreground shell). A Docker daemon restart will revive the container but will not re-launch ad-hoc foreground processes.

---

# Standalone Recipes

## Recipe A — Deploy a Node.js App Behind Nginx

```bash
# On the VPS, as deploy user:
git clone <your-repo> /home/deploy/app
cd /home/deploy/app
nvm use --lts
npm install
npm run build   # if SPA / frontend

# Process management
npm install -g pm2
pm2 start npm --name myapp -- start
pm2 save
pm2 startup systemd   # follow the printed instructions

# Nginx reverse proxy
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

# SSL
# sudo certbot --nginx -d yourdomain.com
```

## Recipe B — Docker Compose Stack

```bash
# Locally: write docker-compose.yml. Then:
scp docker-compose.yml deploy@<VPS-IP>:/home/deploy/
ssh deploy@<VPS-IP>
cd /home/deploy
docker compose up -d
docker compose logs -f
```

**For each port the stack publishes, decide its bucket.** Internal-only services (databases, caches, internal APIs) should publish to `127.0.0.1:port:port`. Public services should be behind a reverse proxy (a Traefik or Nginx container in the same compose file).

## Recipe C — Quick Fix: Server Running Out of Disk

```bash
df -h
du -sh /var/log/* 2>/dev/null | sort -rh | head -10

# Clean Docker (be careful — removes unused images/containers/networks)
docker system prune -af

# Clean apt
sudo apt clean && sudo apt autoremove -y

# Logs
sudo logrotate -f /etc/logrotate.conf
sudo journalctl --vacuum-time=3d

# Visual exploration
sudo apt install -y ncdu && ncdu /
```

## Recipe D — Reset Server to Clean State

**DESTRUCTIVE — read every line before running any.** Take a provider snapshot first. Every command is commented out; uncomment only what you intend.

```bash
# Remove all Docker resources (containers, images, volumes — irreversible)
# docker system prune -a --volumes -f

# Kill all pm2 processes
# pm2 kill

# Remove app data, preserve dotfiles
# rm -rf /home/deploy/app /home/deploy/agents

# Full service uninstall (most destructive — affects other users of these services)
# sudo apt purge -y docker-ce nginx postgresql redis-server
```

---

## Detailed Troubleshooting

| Problem | Likely Cause | Fix |
|---|---|---|
| `ssh: connect to host port 22: Connection refused` | VPS still booting | Wait 30s, retry; check provider dashboard |
| `permission denied (publickey)` | SSH key not deployed | Reconnect with password first, then `ssh-copy-id` |
| `sudo: unable to resolve host` | `/etc/hosts` entry missing | `echo "127.0.1.1 $(hostname)" | sudo tee -a /etc/hosts` |
| `docker: permission denied` | User not yet in docker group | `exit` and SSH back in (group changes apply only on new login) |
| Nginx `address already in use` | Apache/other on port 80 | `sudo lsof -i :80`; stop the conflicting service |
| certbot `too many certificates` | Let's Encrypt rate limit (50/week) | Test with `--staging` first |
| `No space left on device` | Docker/logs filling disk | Recipe C |
| `Python 3.12 not found` | pyenv not in PATH | `source ~/.bashrc` or `eval "$(pyenv init -)"` |
| Container running but service inside dead | Docker daemon restarted; foreground gateway/process inside not auto-restarted | `docker restart <container>`; long-term, run the inner service via the container's entrypoint or a supervisor |
