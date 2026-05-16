# agent-skill-vps-kit 🚀

An AI-agent-readable skill for provisioning a fresh VPS from scratch.

**Works with:** Hermes, Claude Code, Codex, OpenCode, OpenClaw — any AI agent that can read structured markdown.

## What's Inside

| Skill | Description | Version |
|---|---|---|
| [vps-provisioning](./skills/vps-provisioning/SKILL.md) | Full VPS bootstrap: SSH hardening, Docker, Nginx, Node.js, Python, PostgreSQL, Redis, SSL, Netdata | 1.1.0 |

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
