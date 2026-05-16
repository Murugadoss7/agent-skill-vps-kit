     1|---
     2|name: vps-provisioning
     3|description: "Use when provisioning a fresh VPS, installing server software, or setting up an AI-agent host. Covers Ubuntu/Debian: SSH, Docker, Nginx, Node.js, Python, PostgreSQL, Redis, SSL, monitoring, and security."
     4|version: 1.0.1
     5|author: Murugadoss
     6|license: MIT
     7|# marketplace: ai-agent-skills
     8|# id: skill-vps-provisioning
     9|platforms: [linux, macos]
    10|metadata:
    11|  hermes:
    12|    tags: [vps, devops, provisioning, server, docker, nginx, security, deployment]
    13|    related_skills: [claude-code, codex, hermes-agent]
    14|  marketplace:
    15|    category: DevOps
    16|    price: free
    17|    requires: [ssh-client, terminal]
    18|    agent_types: [hermes, claude-code, codex, opencode, openclaw]
    19|---
    20|
    21|# VPS Provisioning & Installation
    22|
    23|An AI-agent-readable guide for provisioning a fresh VPS from scratch. Every recipe is a self-contained workflow an agent can execute step-by-step.
    24|
    25|## When to Use
    26|
    27|- You have a **fresh Ubuntu/Debian VPS** that needs bootstrapping (Docker, Nginx, Node.js, etc.)
    28|- You need to **reprovision** a server to a known state after a reset
    29|- You're setting up a **host for AI agents** (OpenClaw, Hermes gateway, MCP servers)
    30|- You need to **install a specific stack** (LAMP, JAMstack, ML inference server)
    31|
    32|**Don't use for:** Windows Server, Alpine Linux (packages differ significantly), or already-provisioned servers where you just need one package (use `apt` directly).
    33|
    34|## Prerequisites
    35|
    36|| Requirement | Check |
    37||---|---|
    38|| VPS IP address or hostname | `ping <vps-ip>` responds |
    39|| Root password **or** SSH key deployed | Ask provider for root access |
    40|| SSH client installed | `which ssh` |
    41|| DNS (optional) | A record pointing to VPS IP |
    42|| This session uses `pty=true` | For interactive SSH prompts |
    43|
    44|## Quick Start: Full Bootstrap (15 min)
    45|
    46|Run these in order on a fresh VPS. Each section is idempotent — safe to re-run.
    47|
    48|### 0. Distro Detection — Check Before You Start
    49|
    50|```bash
    51|# Run on VPS immediately after first SSH login
    52|cat /etc/os-release | grep -E '^ID=|^VERSION_ID='
    53|```
    54|
    55|Write down the values — several commands in this skill differ between **Ubuntu 24.04+** and older releases (most notably the SSH service name). If `VERSION_ID` is 24.04 or higher, make a mental note: `ssh` not `sshd`.
    56|
    57|### 1. Initial SSH Access & User Setup
    58|
    59|```bash
    60|# Connect as root (you'll be prompted for password)
    61|ssh root@<VPS-IP>
    62|
    63|# Create a deploy user with sudo
    64|adduser deploy
    65|usermod -aG sudo deploy
    66|
    67|# Copy your local SSH key to the deploy user
    68|# Run on YOUR machine (not the VPS):
    69|ssh-copy-id deploy@<VPS-IP>
    70|
    71|# Now reconnect as deploy
    72|ssh deploy@<VPS-IP>
    73|```
    74|
    75|**Pitfall:** If `ssh-copy-id` isn't available, manually:
    76|```bash
    77|# On your machine:
    78|cat ~/.ssh/id_ed25519.pub | ssh deploy@<VPS-IP> "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys && chmod 700 ~/.ssh"
    79|```
    80|
    81|### 2. System Update & Hardening
    82|
    83|```bash
    84|# Run as root or with sudo
    85|sudo apt update && sudo apt upgrade -y
    86|sudo apt install -y ufw fail2ban unattended-upgrades curl wget git htop net-tools
    87|
    88|# Configure firewall
    89|sudo ufw default deny incoming
    90|sudo ufw default allow outgoing
    91|sudo ufw allow ssh
    92|sudo ufw allow http
    93|sudo ufw allow https
    94|sudo ufw --force enable
    95|
    96|# Harden SSH
    97|sudo sed -i 's/^#PermitRootLogin yes/PermitRootLogin prohibit-password/' /etc/ssh/sshd_config
    98|sudo sed -i 's/^#PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config
    99|sudo sed -i 's/^PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config
   100|# Restart SSH — service name differs by distro version
   101|# Ubuntu 24.04+ uses `ssh`, older uses `sshd`
   102|if systemctl list-units --type=service | grep -q ssh.service; then
   103|  sudo systemctl restart ssh
   104|elif systemctl list-units --type=service | grep -q sshd.service; then
   105|  sudo systemctl restart sshd
   106|else
   107|  echo "WARNING: Could not find ssh or sshd service"
   108|fi
   109|
   110|# Enable auto security updates
   111|sudo dpkg-reconfigure --priority=low unattended-upgrades
   112|```
   113|
   114|**Danger zone:** Disabling password auth locks you out if your SSH key isn't deployed. Verify `ssh deploy@<VPS-IP}` works WITHOUT a password BEFORE disconnecting.
   115|
   116|### 3. Swap (Low-RAM VPS)
   117|
   118|```bash
   119|# 2GB swap — adjust size as needed
   120|sudo fallocate -l 2G /swapfile
   121|sudo chmod 600 /swapfile
   122|sudo mkswap /swapfile
   123|sudo swapon /swapfile
   124|echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
   125|# Set swappiness to 10 (default 60, lower = less swapping)
   126|echo 'vm.swappiness=10' | sudo tee -a /etc/sysctl.conf
   127|sudo sysctl -p
   128|```
   129|
   130|### 4. Docker & Docker Compose
   131|
   132|```bash
   133|# Install Docker (official method)
   134|curl -fsSL https://get.docker.com -o get-docker.sh
   135|sudo sh get-docker.sh
   136|
   137|# Add deploy user to docker group (avoids sudo each time)
   138|sudo usermod -aG docker $USER
   139|
   140|# Install Docker Compose plugin
   141|sudo apt install -y docker-compose-plugin
   142|
   143|# Test
   144|docker --version
   145|docker compose version
   146|docker run hello-world
   147|```
   148|
   149|**Pitfall:** The group change only takes effect in a new login shell. After `sudo usermod -aG docker`, log out and back in: `exit` then `ssh deploy@<VPS-IP>`. Don't test `docker ps` before reconnecting — it will fail and confuse you.
   150|
   151|### 5. Nginx + Certbot (SSL)
   152|
   153|```bash
   154|sudo apt install -y nginx certbot python3-certbot-nginx
   155|
   156|# Start and enable
   157|sudo systemctl enable nginx
   158|sudo systemctl start nginx
   159|
   160|# Verify
   161|curl localhost | grep nginx  # Should show "Welcome to nginx"
   162|
   163|# Get SSL (requires DNS pointing here)
   164|# sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com
   165|```
   166|
   167|### 6. Node.js (via nvm)
   168|
   169|```bash
   170|# Install nvm
   171|curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
   172|
   173|# Reload shell config
   174|export NVM_DIR="$HOME/.nvm"
   175|[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
   176|
   177|# Install latest LTS
   178|nvm install --lts
   179|nvm use --lts
   180|node --version  # Should be 22.x or higher
   181|npm --version
   182|
   183|# Optional: pnpm for faster installs
   184|npm install -g pnpm
   185|```
   186|
   187|### 7. Python (pyenv + uv)
   188|
   189|```bash
   190|# Install build dependencies
   191|sudo apt install -y build-essential libssl-dev zlib1g-dev \
   192|  libbz2-dev libreadline-dev libsqlite3-dev libncursesw5-dev \
   193|  xz-utils tk-dev libxml2-dev libxmlsec1-dev libffi-dev liblzma-dev
   194|
   195|# Install pyenv
   196|curl https://pyenv.run | bash
   197|
   198|# Add to shell
   199|echo 'export PYENV_ROOT="$HOME/.pyenv"' >> ~/.bashrc
   200|echo 'command -v pyenv >/dev/null || export PATH="$PYENV_ROOT/bin:$PATH"' >> ~/.bashrc
   201|echo 'eval "$(pyenv init -)"' >> ~/.bashrc
   202|source ~/.bashrc
   203|
   204|# Install Python 3.12 (latest stable)
   205|pyenv install 3.12
   206|pyenv global 3.12
   207|
   208|# Install uv (fast Python package manager)
   209|curl -LsSf https://astral.sh/uv/install.sh | sh
   210|source ~/.bashrc
   211|uv --version
   212|```
   213|
   214|### 8. PostgreSQL
   215|
   216|```bash
   217|sudo apt install -y postgresql postgresql-contrib
   218|
   219|# Start and enable
   220|sudo systemctl enable postgresql
   221|sudo systemctl start postgresql
   222|
   223|# Create a deploy database user
   224|sudo -u postgres createuser --interactive
   225|# Answer: n (not superuser), n (no create db), n (no create roles)
   226|
   227|# Or non-interactively:
   228|sudo -u postgres psql -c "CREATE USER deploy WITH PASSWORD 'generate-a-strong-password';"
   229|sudo -u postgres psql -c "CREATE DATABASE myapp OWNER deploy;"
   230|
   231|# Test
   232|sudo -u postgres psql -c "\l"
   233|```
   234|
   235|### 9. Redis
   236|
   237|```bash
   238|sudo apt install -y redis-server
   239|
   240|# Configure for systemd
   241|sudo sed -i 's/^supervised no/supervised systemd/' /etc/redis/redis.conf
   242|
   243|# Secure with password
   244|# sudo sed -i 's/^# requirepass foobared/requirepass <strong-password>/' /etc/redis/redis.conf
   245|
   246|sudo systemctl enable redis-server
   247|sudo systemctl start redis-server
   248|
   249|# Test
   250|redis-cli ping  # Should return PONG
   251|```
   252|
   253|### 10. Monitoring Stack (Netdata)
   254|
   255|```bash
   256|# One-line Netdata install
   257|bash <(curl -Ss https://my-netdata.io/kickstart.sh)
   258|
   259|# Access: http://<VPS-IP>:19999 (restrict with UFW for safety)
   260|sudo ufw allow 19999/tcp
   261|```
   262|
   263|### 11. AI Agent Host Setup (OpenClaw / Hermes gateway)
   264|
   265|```bash
   266|# For OpenClaw (or any Python-based agent running as a service):
   267|mkdir -p /home/deploy/agents
   268|cd /home/deploy/agents
   269|
   270|# Create a systemd service template
   271|sudo tee /etc/systemd/system/agent.service << 'SERVICEEOF'
   272|[Unit]
   273|Description=AI Agent Service
   274|After=network.target
   275|
   276|[Service]
   277|Type=simple
   278|User=deploy
   279|WorkingDirectory=/home/deploy/agents
   280|Environment="PATH=/home/deploy/.local/bin:/usr/local/bin:/usr/bin:/bin"
   281|ExecStart=/usr/bin/python3 -m <your-agent-module>
   282|Restart=always
   283|RestartSec=5
   284|
   285|[Install]
   286|WantedBy=multi-user.target
   287|SERVICEEOF
   288|
   289|sudo systemctl daemon-reload
   290|```
   291|
   292|## Recipes (Standalone)
   293|
   294|### Recipe A: Deploy a Node.js App Behind Nginx
   295|
   296|```bash
   297|# 1. Build and test the app locally first
   298|# 2. On the VPS:
   299|git clone <your-repo> /home/deploy/app
   300|cd /home/deploy/app
   301|nvm use --lts
   302|npm install
   303|npm run build  # If SPA/frontend
   304|
   305|# 3. PM2 for process management
   306|npm install -g pm2
   307|pm2 start npm --name "myapp" -- start
   308|pm2 save
   309|pm2 startup systemd  # Follow the instructions it prints
   310|
   311|# 4. Nginx reverse proxy config
   312|sudo tee /etc/nginx/sites-available/myapp << 'NGINXEOF'
   313|server {
   314|    listen 80;
   315|    server_name yourdomain.com;
   316|
   317|    location / {
   318|        proxy_pass http://localhost:3000;
   319|        proxy_http_version 1.1;
   320|        proxy_set_header Upgrade $http_upgrade;
   321|        proxy_set_header Connection 'upgrade';
   322|        proxy_set_header Host $host;
   323|        proxy_set_header X-Real-IP $remote_addr;
   324|        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
   325|        proxy_set_header X-Forwarded-Proto $scheme;
   326|        proxy_cache_bypass $http_upgrade;
   327|    }
   328|}
   329|NGINXEOF
   330|
   331|sudo ln -sf /etc/nginx/sites-available/myapp /etc/nginx/sites-enabled/
   332|sudo nginx -t && sudo systemctl reload nginx
   333|
   334|# 5. SSL
   335|# sudo certbot --nginx -d yourdomain.com
   336|```
   337|
   338|### Recipe B: Docker Compose Stack
   339|
   340|```bash
   341|# On your machine, write a docker-compose.yml, then:
   342|scp docker-compose.yml deploy@<VPS-IP>:/home/deploy/
   343|ssh deploy@<VPS-IP>
   344|cd /home/deploy
   345|docker compose up -d
   346|docker compose logs -f
   347|```
   348|
   349|Apply for: Databases, message queues, monitoring (Grafana/Prometheus), full-stack apps.
   350|
   351|### Recipe C: Quick Fix — Server Running Out of Disk
   352|
   353|```bash
   354|# Check usage
   355|df -h
   356|du -sh /var/log/* | sort -rh | head -10
   357|
   358|# Clean Docker
   359|docker system prune -af  # Removes unused images, containers, networks
   360|
   361|# Clean apt cache
   362|sudo apt clean
   363|sudo apt autoremove -y
   364|
   365|# Rotate and compress logs
   366|sudo logrotate -f /etc/logrotate.conf
   367|sudo journalctl --vacuum-time=3d
   368|
   369|# Check what's taking space
   370|ncdu /  # Install with: sudo apt install -y ncdu
   371|```
   372|
   373|### Recipe D: Reset Server to Clean State
   374|
   375|```bash
   376|# Remove all Docker resources
   377|docker system prune -a --volumes -f
   378|
   379|# Remove all pm2 processes
   380|pm2 kill
   381|
   382|# Remove app data but keep services
   383|rm -rf /home/deploy/*
   384|
   385|# Or full uninstall of services (USE WITH CAUTION):
   386|# sudo apt purge -y docker-ce nginx postgresql redis-server
   387|```
   388|
   389|## Troubleshooting Guide
   390|
   391|| Problem | Likely Cause | Fix |
   392||---|---|---|
   393|| `ssh: connect to host port 22: Connection refused` | VPS still booting | Wait 30s, retry. Check provider dashboard |
   394|| `permission denied (publickey)` | SSH key not deployed | Reconnect as root with password, run `ssh-copy-id` |
   395|| `sudo: unable to resolve host` | /etc/hosts entry wrong | `echo "127.0.1.1 $(hostname)" \| sudo tee -a /etc/hosts` |
   396|| `docker: permission denied` | User not in docker group | Re-login shell: `exit` then SSH back in |
   397|| Nginx `address already in use` | Apache/other on port 80 | `sudo lsof -i :80` then stop the other service |
   398|| certbot `too many certificates` | Rate-limited | Certbot rate limit: 50/week. Use staging: `--staging` for testing |
   399|| `No space left on device` | Docker/logs filling disk | Run Recipe C above |
   400|| `Python 3.12 not found` | pyenv not in PATH | Source `~/.bashrc` or run `eval "$(pyenv init -)"` |
   401|
   402|## Common Pitfalls
   403|
   404|1. **SSH disconnection during long installs.** Use `screen` or `tmux` before starting: `sudo apt install -y tmux && tmux new -s provision`. If disconnected, reconnect and `tmux attach -t provision`.
   405|
   406|2. **SSH service name varies by Ubuntu version.** Ubuntu 24.04+ uses `ssh.service`; older releases use `sshd.service`. If `sudo systemctl restart sshd` fails with `Unit sshd.service not found`, try `sudo systemctl restart ssh` instead. The skill's hardening section now includes a portable auto-detect, but if you're using a cloud-init script or one-liner, hardcode the right name for your target distro.
   407|
   408|3. **Assuming `sudo` without password.** Some VPS images configure sudo to ask for password. Verify: `sudo -n whoami`. If it asks, wait for it — don't panic.
   409|
   410|4. **Docker group doesn't take effect immediately.** The group membership changes apply to the next login shell. Run `newgrp docker` in the same session, or `exit` and re-SSH. Running `docker ps` immediately after `usermod` will fail.
   411|
   412|5. **UFW blocks your SSH before you enable it.** The `sudo ufw allow ssh` rule (or `allow 22`) MUST come before `ufw --force enable`. Most guides get this right, but if you're editing rules remotely, always leave a second SSH session open as a safety net.
   413|
   414|5. **Pyenv builds fail on minimal images.** Missing build deps (Section 7). Install them all before running `pyenv install`.
   415|
   416|6. **Certbot DNS validation fails.** Make sure the A record points to your VPS IP and has propagated. Check with `dig yourdomain.com +short`.
   417|
   418|7. **Container networking conflicts.** If Docker containers can't reach each other, check `docker network ls` and ensure they share a network: `docker network create mynet` and reference it in compose.
   419|
   420|8. **Port already in use after reboot.** Services with `Restart=always` (Docker, PM2) restart and reclaim ports faster than Nginx. Set `After=network.target docker.service` in Nginx systemd override, or use `restart: unless-stopped` in compose.
   421|
   422|## Verification Checklist
   423|
   424|After a full bootstrap, run through this:
   425|
   426|- [ ] `ssh deploy@<VPS-IP>` — connects without password
   427|- [ ] `sudo -n whoami` — returns `root` without prompting
   428|- [ ] `ufw status verbose` — shows `22/tcp`, `80/tcp`, `443/tcp` ALLOW
   429|- [ ] `docker run --rm hello-world` — Docker works without sudo
   430|- [ ] `curl localhost | grep nginx` — Nginx welcome page
   431|- [ ] `node --version` — ≥ 22.x
   432|- [ ] `python3 --version` — ≥ 3.12
   433|- [ ] `psql --version` — PostgreSQL client works
   434|- [ ] `redis-cli ping` — returns PONG
   435|- [ ] `pm2 list` — PM2 available (if installed)
   436|- [ ] `systemctl status netdata | grep active` — Netdata running
   437|- [ ] `df -h | grep /dev/sda` — < 60% disk usage
   438|- [ ] `free -h | grep Swap` — swap active
   439|- [ ] Browser: `http://<VPS-IP>` — Nginx landing page
   440|- [ ] DNS (if set): `dig yourdomain.com +short` — resolves to VPS IP
   441|
   442|## Reference Files
   443|
   444|- `references/ubuntu-24-differences.md` — Ubuntu 24.04 quirks (ssh vs sshd, network interfaces, Python, Docker, AppArmor)
   445|
   446|## Marketplace Info
   447|
   448|**Skill ID:** `vps-provisioning`
   449|**Category:** DevOps / Server Management
   450|**Agent compatibility:** Hermes, Claude Code, Codex, OpenCode, OpenClaw, any AI agent that reads structured markdown
   451|**Dependencies:** `ssh`, `terminal` (any shell) — no proprietary APIs
   452|**Upgrade path:** Append recipes to the skills directory, version bump
   453|
   454|---
   455|
   456|*Built for the Agent Skills Marketplace. Ship fast, iterate in the open.*
   457|