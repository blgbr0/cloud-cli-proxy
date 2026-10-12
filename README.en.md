<div align="center">

<img src="docs/public/logo.svg" width="88" height="88" alt="Cloud CLI Proxy" />

# Cloud CLI Proxy

**Enterprise-grade secure remote development environment. A containerized, isolated, auditable, and policy-driven platform for running Claude Code in teams.**

[![CI](https://github.com/ZaneL1u/cloud-cli-proxy/actions/workflows/ci.yml/badge.svg)](https://github.com/ZaneL1u/cloud-cli-proxy/actions/workflows/ci.yml)
[![Images](https://github.com/ZaneL1u/cloud-cli-proxy/actions/workflows/build-images.yml/badge.svg)](https://github.com/ZaneL1u/cloud-cli-proxy/actions/workflows/build-images.yml)
[![Release](https://img.shields.io/github/v/release/ZaneL1u/cloud-cli-proxy)](https://github.com/ZaneL1u/cloud-cli-proxy/releases)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

[中文](README.md) | [Docs](https://zanel1u.github.io/cloud-cli-proxy/en/)

</div>

---

## The Problem

As AI coding tools become widespread in enterprises, security and compliance teams face new challenges: inconsistent development environments, excessive AI agent permissions, sensitive codebases lacking isolation, unmanaged network egress, and unauditable user actions. Running Claude Code directly on production systems or personal devices often fails to meet enterprise security baselines and compliance requirements.

Cloud CLI Proxy provides a **self-hosted, containerized remote development infrastructure** that encapsulates Claude Code in a controlled, isolated environment. Each developer gets an independent, standardized container workspace. All network egress, system configurations, and user actions are governed by platform-wide policies, ensuring AI coding activities operate within a **secure, compliant, and auditable** framework.

Deploy it on your own server. Developers SSH in and start coding. Local project directories mount into the container with identical paths, delivering a development experience indistinguishable from running locally.

---

## How It Works

### Environment Isolation & Security Hardening

The platform standardizes and isolates development environments across multiple dimensions, ensuring container workspaces are decoupled from the host system and meet enterprise security baselines:

- **System environment standardization** — Unifies CPU model, MAC address, machine-id, and other system identifiers, ensuring consistent container environments for all developers and eliminating security blind spots caused by environment drift. Outputs from `ioreg`, `system_profiler`, `sysctl` are policy-processed to avoid leaking underlying infrastructure details.
- **Standardized hostnames** — Auto-generates hostnames following enterprise naming conventions for asset management and audit trails.
- **Minimized attack surface** — Removes unnecessary system information exposure inside containers (e.g., `/.dockerenv`, container runtime identifiers in cgroup), following security hardening best practices to reduce potential attack surface.
- **Timezone & locale compliance** — Defaults to Pacific Time and `en_US.UTF-8`, adjustable per container to meet regional compliance and localization requirements.
- **Outbound traffic compliance** — Outbound connections use uTLS with a standard Chrome fingerprint, ensuring network communications align with enterprise egress policies and avoiding anomalous traffic patterns that trigger security alerts.
- **Data Loss Prevention (DLP) & network egress control** — Implements DNS-level network policies to block telemetry and tracking domains such as `statsig.anthropic.com`, `sentry.io`, and `cdn.growthbook.io`, preventing accidental data exfiltration through AI agents and meeting enterprise DLP requirements.

### Network Policy Enforcement & Egress Isolation

Every container is bound to an independent egress IP. All traffic — HTTP, DNS, WebRTC — exits through a sing-box TUN tunnel from that designated IP. nftables default-deny ensures no policy bypass paths exist.

Six proxy protocols are supported: SOCKS5, VMess, VLESS, Shadowsocks, Trojan, HTTP — making it easy to integrate with existing enterprise network egress or compliant proxy infrastructure.

**Enterprise egress recommendation**: Use network egress that meets your enterprise compliance requirements, such as corporate-owned residential broadband, commercial VPN, or cloud network egress. Avoid flagged data center IP ranges to ensure stability and compliance of network policies.

### Code Mapping — Consistent Local Development Experience

This is what separates it from a regular VPS. The container mounts your local project directory via sshfs at the **exact same path**.

Working in `~/my-project` locally? It's `~/my-project` inside the container too. Claude Code sees paths that match your machine perfectly — just like running natively. Three mount modes: full sync, smart hot-sync, and plain mount. Disconnects auto-recover within 30 seconds, input buffer survives. tmux sessions keep your workspace alive across disconnects.

### Zero Friction

Admin creates the container from the dashboard. You get a `curl` command. Paste it, enter a password, wait a few seconds for the container to boot, and you're SSHed in. Claude Code is pre-installed. Just type `claude`.

---

## Quick Start

```bash
git clone https://github.com/ZaneL1u/cloud-cli-proxy.git
cd cloud-cli-proxy

bash deploy/scripts/setup-env.sh

docker compose pull
docker compose up -d

curl http://127.0.0.1:8080/healthz
# {"status":"ok"}
```

After startup:

- Admin dashboard: `http://YOUR_HOST:8080`
- API: `http://YOUR_HOST:8080`
- SSH proxy: `YOUR_HOST:2222`

First-time setup: log into admin dashboard → add egress IPs → create users → create hosts → share the access command.

---

## Deployment

### Requirements

- Docker Engine 28.x+
- Docker Compose v2

### Docker Compose (recommended)

```bash
bash deploy/scripts/setup-env.sh  # Interactive setup
docker compose pull               # Pull prebuilt images
docker compose up -d              # Start
```

### Bare-metal

```bash
sudo bash deploy/scripts/deploy.sh
```

Creates a `cloudproxy` system user, builds binaries and images, installs systemd units, starts services.

### Environment Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `DATABASE_URL` | SQLite database path | `file:/data/cloud-cli-proxy.db` |
| `ADMIN_USERNAME` | Admin username | `admin` |
| `ADMIN_PASSWORD` | Admin password (bcrypt) | Required |
| `ADMIN_JWT_SECRET` | JWT signing secret | Required |
| `SSH_PROXY_PORT` | SSH proxy port | `2222` |
| `LOG_FORMAT` | Log format `json` / `text` | `json` |
| `LOG_LEVEL` | Log level | `info` |

---

## Admin Dashboard

Beyond environment isolation and network policy enforcement, the platform includes full governance:

- User lifecycle — create, suspend, auto-expire, password rotation, RBAC support
- Host lifecycle — create, start, stop, rebuild (keep or wipe `/workspace`), delete
- Egress IP management — CRUD, connectivity testing
- Bypass firewall — domain, CIDR, port whitelisting, snapshot versioning, preview → apply → rollback
- Full audit trail — every operation written to the events table, supporting compliance audits (SOX, GDPR, etc.)
- SSE real-time push for task progress and host status
- Built-in KasmVNC + Chromium — one click from dashboard to remote desktop

---

## Architecture

```
                                                    ┌───────────────────────────────────┐
User ──curl──> Control Plane (:8080) ──Docker──>    │ User Container                    │
                    │                                │  SSH + Claude Code + VNC          │
               SQLite                              │  sshfs ← same path as local cwd  │
                    │                                │  sing-box tun tunnel              │
              Admin UI (embed)                      │       ↓                           │
                    │                                │  Designated Exit IP               │
              SSH Proxy (:2222)                      └───────────────────────────────────┘
```

| Component | Description |
|-----------|-------------|
| **Control Plane** | Go API — auth, user management, task orchestration, SSH proxy |
| **Host Agent** | Privileged agent — manages Docker containers, network namespaces, and tunnels |
| **User Container** | Ubuntu 24.04 — OpenSSH + Claude Code + sshfs + KasmVNC + Chromium |
| **SQLite** | Persists users, hosts, egress IPs, tasks, events, audit logs (WAL mode) |
| **Admin SPA** | React 19 + TypeScript + Vite + Tailwind CSS |

---

## Compliance & Legal Notice

- This project is designed as self-hosted enterprise infrastructure to meet security, audit, and compliance requirements.
- Each end user should authenticate with their own credentials and comply with Anthropic's commercial terms of service.
- This project does not provide, encourage, or support any behavior intended to circumvent terms of service or bypass security mechanisms. Users are responsible for ensuring their usage complies with applicable laws and service provider policies.
- Released under the MIT License. The authors assume no liability for any legal or compliance issues arising from the use of this project.

---

## Contributing

Bug reports and feature requests: open an [Issue](https://github.com/ZaneL1u/cloud-cli-proxy/issues).

Pull request process:

1. Fork the repo, create a feature branch from `main`
2. Make your changes, ensure `make test` passes
3. Open a PR describing what you changed and why

Local dev environment:

```bash
make setup    # Install dependencies
make dev      # Backend + frontend hot-reload (API :8080, frontend localhost:2568)
make test     # Run tests
```

See `make help` for all commands.

---

## Documentation

Full docs on [GitHub Pages](https://zanel1u.github.io/cloud-cli-proxy/en/): quick start, deployment, configuration, architecture, API reference, troubleshooting.

---

## License

[MIT](LICENSE)
