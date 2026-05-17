# agent-skill-vps-kit 🚀

An AI-agent-readable skill for VPS **hardening, auditing, and provisioning** on Ubuntu/Debian.

**Works with:** Hermes, Claude Code, Codex, OpenCode, OpenClaw — any AI agent that can read structured markdown.

## What's Inside

| Skill | Description | Version |
|---|---|---|
| [vps-provisioning](./skills/vps-provisioning/SKILL.md) | Two paths: (A) audit & harden an existing server running Docker/apps without disrupting them; (B) bootstrap a fresh empty VPS (Docker, Nginx, Node, Python, Postgres, Redis, SSL, monitoring). Teaches default-deny UFW, port classification, and the Docker-bypasses-UFW gotcha. | 1.2.0 |

## Quick Start

Load the skill into your AI agent and run:

```bash
# Every recipe is self-contained and idempotent
# Start with SSH key setup, then work through the sections in order
```

See the [skill file](./skills/vps-provisioning/SKILL.md) for full instructions.

## Roadmap

- [ ] More recipes (LAMP stack, ML inference server, MCP server hosting)
- [ ] Distro-specific variants (Alpine, CentOS)
- [ ] Automated test suite

## License

MIT
