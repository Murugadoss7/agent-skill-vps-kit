---
name: vps-provisioning
description: "Use when hardening, auditing, or provisioning an Ubuntu/Debian VPS. Two paths: (A) audit & harden an existing server that already runs Docker, applications, or reverse proxies — discovers what's listening, classifies ports as public/private/loopback, applies safe baseline hardening without disrupting running services; (B) bootstrap a fresh empty host with Docker, Nginx, Node, Python, Postgres, Redis, SSL, monitoring. Explains the Docker-bypasses-UFW gotcha. Loads Path B from references/bootstrap-fresh.md only when the host is confirmed empty."
license: MIT
compatibility: Requires an SSH client and an Ubuntu/Debian VPS with sudo or root access
metadata:
  author: Murugadoss
  version: "1.2.0"
  tags: "vps devops hardening audit firewall ufw docker security ssh provisioning"
  category: DevOps
---

# VPS Hardening & Provisioning

A two-path guide. **Path A** audits and hardens a host that's already running things (Docker, apps, a reverse proxy) without breaking them. **Path B** bootstraps a fresh empty host into a known stack.

## Choose Your Path

Run this first:

```bash
# Is anything already running that we'd disturb?
docker ps 2>/dev/null | tail -n +2 | wc -l        # number of running containers
systemctl --no-pager list-units --type=service --state=running \
  | grep -cE 'nginx|apache|postgresql|redis|caddy|traefik|haproxy' || true
ss -tlnp 2>/dev/null | tail -n +2 | wc -l         # number of listening TCP sockets
```

- **Containers or matching services > 0**, or many listening sockets → **use Path A**. Running Path B's installer steps (especially Docker re-install or Nginx install) will restart the Docker daemon and may break running containers/proxies.
- **All zeros, fresh VPS** → **use Path B**.

## What "Hardening" Actually Means (read once, applies to both paths)

Default-deny + explicit allow-list is the only real firewall posture. The threat: anything you accidentally bind to `0.0.0.0` is exposed. With UFW default-deny, only ports you've **explicitly** allowed respond; the allow-list becomes your security inventory.

There is **no universal port list.** SSH (22) is the only one everyone needs. After that, you allow what *your* services use — not "80/443 because LAMP."

Classify every service into one of three buckets:

| Bucket | When | How |
|---|---|---|
| **Public** | Web traffic that the internet must reach (reverse proxy on 443, public API) | `sudo ufw allow <port>/tcp` |
| **Private** | Admin or internal services that only a known office/VPN IP should reach | `sudo ufw allow from <ip-or-cidr> to any port <port>` |
| **Loopback / internal-only** | Databases, gateways, dashboards that should never be reached from outside | Bind the service to `127.0.0.1` and **don't** open the port in UFW |

### Docker bypasses UFW — this is the most-missed footgun

When you `docker run -p 18789:18789`, Docker writes iptables rules at a higher priority than UFW. The port is **publicly reachable** even when `ufw status` does not list it. Three fixes:

1. **Make it loopback-only:** `-p 127.0.0.1:18789:18789` instead of `-p 18789:18789`. UFW still doesn't apply, but the port is only reachable from the host. Pair with a reverse proxy (Traefik, Nginx) for internet access.
2. **Use an internal Docker network:** no `-p` flag at all. Other containers on the same network can reach it; the host and the internet cannot.
3. **Reverse proxy in `--network host` mode:** as Traefik commonly does. Then UFW *does* control it.

The skill enforces this in Path A by listing every Docker-published port that's currently `0.0.0.0:port:port` and asking the operator: "should this be public, private, or loopback?"

---

# PATH A — Audit & Harden an Existing Server

Safe on hosts already running Docker, web apps, databases, reverse proxies. Adds no new services. Does not restart Docker. Does not install Nginx.

## A.1 Pre-flight Audit

Run all of these and **save the output**. Subsequent steps reference it.

```bash
# What containers exist, with their published ports
docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}' 2>/dev/null

# What's listening on the host (both host services AND Docker port mappings)
sudo ss -tlnp 2>/dev/null || sudo netstat -tlnp

# What systemd services are running
systemctl --no-pager --type=service --state=running

# Distro check — several commands differ between Ubuntu 24.04+ and older
cat /etc/os-release | grep -E '^ID=|^VERSION_ID='

# Current UFW state
sudo ufw status verbose 2>/dev/null || echo "UFW not installed"
```

For each TCP port in `ss -tlnp` output, decide its bucket (public / private / loopback) using the table above. Write it down. You will need this list in step A.4.

## A.2 SSH Hardening (safe on any server)

Adds an `sshd_config.d` drop-in. Does not touch the main config file. Validates before restart. Survives package upgrades.

```bash
# Drop-in config (single-line printf — heredocs break when wrapped in `ssh user@host "..."`)
sudo install -d -m 0755 /etc/ssh/sshd_config.d
printf '%s\n' \
  'PermitRootLogin prohibit-password' \
  'PasswordAuthentication no' \
  'KbdInteractiveAuthentication no' \
  | sudo tee /etc/ssh/sshd_config.d/10-hardening.conf > /dev/null

# Validate BEFORE restart
sudo sshd -t || { echo "ABORT: sshd config invalid"; exit 1; }

# Restart SSH (service name varies)
if systemctl list-unit-files | grep -q '^ssh\.service'; then
  sudo systemctl restart ssh
elif systemctl list-unit-files | grep -q '^sshd\.service'; then
  sudo systemctl restart sshd
fi
```

**Danger zone:** BEFORE you run the block above, open a **second SSH session** and confirm you can log in without a password (`ssh -o PreferredAuthentications=publickey deploy@<VPS-IP>`). Keep it open until a third session also works. `sshd -t` catches syntax errors, not policy mistakes — only a real login proves the policy.

## A.3 Baseline System Hardening (safe on any server)

```bash
# Tools that aren't installs of new long-running services — all safe.
sudo apt update
sudo apt install -y ufw fail2ban unattended-upgrades

# Enable automatic security updates
sudo dpkg-reconfigure --priority=low unattended-upgrades

# fail2ban with defaults protects SSH. Enable + start.
sudo systemctl enable --now fail2ban

# Swap if not present (idempotent)
if [ ! -f /swapfile ] && [ "$(awk '/SwapTotal/ {print $2}' /proc/meminfo)" -lt 524288 ]; then
  sudo fallocate -l 2G /swapfile
  sudo chmod 600 /swapfile
  sudo mkswap /swapfile
  sudo swapon /swapfile
fi
grep -qxF '/swapfile none swap sw 0 0' /etc/fstab 2>/dev/null || \
  echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab > /dev/null
grep -qxF 'vm.swappiness=10' /etc/sysctl.conf || \
  echo 'vm.swappiness=10' | sudo tee -a /etc/sysctl.conf > /dev/null
sudo sysctl -p
```

## A.4 Operator-Driven UFW (the critical step)

**Do NOT just enable UFW with default-deny without doing this first.** You will lock out anything you didn't list. Use your A.1 audit output.

```bash
# Step 1: ALWAYS allow SSH first. Losing this locks you out.
sudo ufw allow OpenSSH

# Step 2: For each service in your audit, decide its bucket and run ONE of:

# (a) Public — internet-facing reverse proxy / public API:
#     sudo ufw allow 443/tcp comment 'Traefik HTTPS'
#     sudo ufw allow 80/tcp  comment 'Traefik HTTP redirect'

# (b) Private — admin UI, internal API, you have a static IP at the office:
#     sudo ufw allow from 203.0.113.5 to any port 19999 proto tcp comment 'Netdata from office'

# (c) Loopback — should never be UFW-allowed. Instead, ensure the service binds
#     only to 127.0.0.1. For Docker, recreate the container with -p 127.0.0.1:PORT:PORT.

# Step 3: Default policies (only AFTER steps 1+2)
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Step 4: Enable. Keep a SECOND SSH session open as a safety net.
sudo ufw --force enable

# Step 5: Verify
sudo ufw status verbose
```

### Docker-vs-UFW reality check

After UFW is enabled, run:

```bash
# Listening sockets vs UFW rules — what's actually reachable?
echo "=== UFW says these are allowed: ==="
sudo ufw status numbered

echo "=== ss -tlnp says these are listening on 0.0.0.0 or ::0 (public IPs): ==="
sudo ss -tlnp 'src 0.0.0.0 or src [::]' | tail -n +2
```

**Any port in the second list that is NOT in the first list is a Docker-published port that UFW cannot block.** To make it private:

```bash
# Find the container that publishes it
docker ps --format 'table {{.Names}}\t{{.Ports}}' | grep <port>

# Demote to loopback (requires recreate; data on named volumes survives)
docker inspect <container> --format '{{.Name}} {{.Config.Image}} {{json .HostConfig.PortBindings}}'
# Then recreate the container with -p 127.0.0.1:<port>:<port> instead of -p <port>:<port>.
# Easiest path: edit docker-compose.yml and `docker compose up -d` to recreate cleanly.
```

For backend services (DBs, caches, gateways), loopback-binding plus a reverse proxy is almost always the right answer. The internet reaches only the proxy; the proxy reaches the backend through the Docker network or 127.0.0.1.

## A.5 Verify

```bash
# A. SSH login works WITHOUT password (in a fresh session)
ssh -o PreferredAuthentications=publickey deploy@<VPS-IP> 'echo OK'

# B. UFW is active and lists exactly what you intended
sudo ufw status verbose

# C. fail2ban is running
sudo systemctl is-active fail2ban

# D. Auto-updates configured
sudo systemctl is-enabled unattended-upgrades

# E. Containers still running, none restart-looping
docker ps --filter status=running
docker ps --filter status=restarting    # should be empty

# F. Reverse proxy still terminates 443
curl -sI https://<your-domain> | head -5

# G. No surprise public ports
sudo ss -tlnp 'src 0.0.0.0 or src [::]' | tail -n +2
```

If every check passes, you're done. The existing stack is now hardened without being disrupted.

---

# PATH B — Bootstrap a Fresh Host

**Pre-flight check (mandatory; refuse to proceed otherwise):**

```bash
# These must ALL be zero or the host is not fresh — use Path A instead.
CONTAINERS=$(docker ps -q 2>/dev/null | wc -l)
SERVICES=$(systemctl --no-pager list-units --type=service --state=running \
  | grep -cE 'nginx|apache|postgresql|redis|caddy|traefik|haproxy|docker' || echo 0)
if [ "$CONTAINERS" -gt 0 ] || [ "$SERVICES" -gt 0 ]; then
  echo "REFUSING: host has running services or containers. Use Path A (harden existing) instead."
  exit 1
fi
echo "OK: host appears empty — proceeding with fresh bootstrap."
```

The detailed fresh-bootstrap recipes (Docker install, Nginx, Node via nvm, Python via pyenv, Postgres, Redis, Netdata, systemd agent template) are in [references/bootstrap-fresh.md](references/bootstrap-fresh.md). Load it only after the pre-flight passes.

The agent setup, deploy-user creation, Recipes A–D (Node app, Docker Compose stack, disk cleanup, reset), and a deeper troubleshooting guide all live in that reference file.

---

## Troubleshooting (applies to both paths)

| Problem | Likely Cause | Fix |
|---|---|---|
| `permission denied (publickey)` after A.2 | Key not actually deployed | Reconnect with password from a session you kept open; re-run `ssh-copy-id` |
| UFW enabled, SSH lost | `ufw allow OpenSSH` not run before `--force enable` | Provider web console → reset rules: `sudo ufw disable` |
| Service unreachable AFTER A.4 | Port not in allow-list, OR Docker bypass needed | Check `ss -tlnp` vs `ufw status`; if Docker, demote to loopback or add explicit allow |
| Mission Control / dashboard 502/timeout | Reverse proxy can't reach backend | Backend container restart-looping after Docker daemon event; `docker logs <container>` |
| `Failed to start: gateway already running; lock timeout` | Stale lock file from prior half-crash | `docker exec <container> rm -f /data/.openclaw/*.lock` then `docker restart` |

## Common Pitfalls

1. **Hardcoding 22/80/443 in UFW.** That's the LAMP-stack assumption. Real apps use other ports. Always start from A.1 audit, never from a generic list.
2. **Forgetting Docker bypasses UFW.** `ufw status` will lie about what's reachable. Always cross-check with `ss -tlnp 'src 0.0.0.0'`.
3. **Running Path B on a host with Docker already running.** `bash get-docker.sh` re-installs and restarts the Docker daemon; any container without `--restart unless-stopped` will stop, and even containers with restart policies will lose any foreground processes started inside them (e.g., a gateway started in tmux). This is what the Path B pre-flight check exists to prevent.
4. **SSH service name varies.** Ubuntu 24.04+ uses `ssh.service`; older uses `sshd.service`. A.2 auto-detects.
5. **Disabling password auth without verifying key login.** Always keep a second SSH session open and verify a third works before disconnecting.
6. **`docker usermod -aG docker $USER` under sudo** adds *root*, not the user. Hardcode the target user.
7. **`curl … | bash` installers** (Docker, nvm, pyenv, uv, Netdata) execute remote scripts. All HTTPS, but contents can change. For agentic execution, pin versions where supported and audit in security-sensitive environments.
8. **Heredocs break when wrapped in `ssh user@host "..."`** because the outer quotes close at the first inner quote. Prefer `printf '%s\n' a b c | tee file` for short configs in agent contexts.

## Compatibility

- **Ubuntu 24.04+** ships SSH as `ssh.service`; older releases use `sshd.service`. The hardening section auto-detects.
- **Debian 12** works identically; package names match.
- **Not supported:** Alpine (apk, OpenRC), CentOS/RHEL (dnf), Windows Server.

## References

- [bootstrap-fresh.md](references/bootstrap-fresh.md) — Detailed install recipes for fresh-host bootstrap (Path B): Docker, Nginx, Node, Python, Postgres, Redis, Netdata, AI agent host, and standalone recipes for Node-app-behind-Nginx, Docker Compose, disk cleanup, and full reset.

---

*Built for the Agent Skills ecosystem. Audit-first. Hardens what's there without breaking what runs.*
