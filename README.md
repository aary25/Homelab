# Homelab

> Self-hosted DevOps stack on Proxmox VE · Debian 13 Trixie · Docker

## Architecture

<!-- Export the architecture diagram as a PNG and drop it in docs/: -->
<!-- ![Architecture](./docs/architecture.png) -->

## Hardware

| Component | Details |
|-----------|---------|
| Host | Dell OptiPlex 7050 SFF |
| CPU | Intel Core i7-6700 (4C/8T, 3.4 GHz) |
| RAM | 24 GB DDR4 |
| Hypervisor | Proxmox VE 8 |
| Network | NETGEAR RAX48 — DHCP MAC reservations for all three devices |

## Virtual machines

| VM | IP | OS | RAM | Role |
|----|----|----|-----|------|
| VM 100 · `debian` | `192.168.1.6` | Debian 13 Trixie | 8 GB | Docker host — all services |
| VM 101 · `minecraft` | `192.168.1.8` | Debian 13 Trixie | 12 GB | Dedicated game server |

Both VMs are set to `onboot: 1` — they auto-start whenever the Proxmox host boots.

## Stack

| Layer | Tool | Purpose |
|-------|------|---------|
| Hypervisor | Proxmox VE 8 | VM provisioning and management |
| Firewall | UFW | Default-deny inbound, explicit port allowlist |
| IDS | Fail2ban | Auto-bans IPs after repeated failed logins |
| VPN | Tailscale | Mesh VPN for remote admin — punches through CG-NAT |
| DNS | DuckDNS (`aarylab.duckdns.org`) | Dynamic hostname updated every 5 min via cron |
| Proxy | Nginx Proxy Manager | Reverse proxy + automatic TLS (Let's Encrypt) |
| Management | Portainer | Container UI and Docker Compose stack management |
| Git | Gitea | Self-hosted version control |
| CI/CD | Woodpecker | Automated pipelines triggered on every Gitea push |
| Metrics | Prometheus + Node Exporter | Scrapes system metrics every 15 s |
| Dashboards | Grafana | Visualization, custom dashboards, threshold alerting |
| Media | Calibre-web | Self-hosted ebook library |
| Game | Minecraft Java (`itzg/minecraft-server`) | Dedicated server with Aikar JVM flags |
| Tunnel | playit.gg | External Minecraft access despite CG-NAT |

## Networking

The apartment ISP uses **CG-NAT** — the WAN IP is in the `10.x.x.x` range, making traditional inbound port forwarding from the router impossible.

Three-part solution:

- **Tailscale** (WireGuard-based mesh VPN): installed on VM 100 and all admin devices. Establishes peer-to-peer tunnels that punch through CG-NAT with no open ports required. VM 100 Tailscale IP: `100.92.79.114`. This handles all remote administration.
- **playit.gg**: a tunnel agent on VM 101 proxies Minecraft traffic through playit's relay servers. Public join address: `physical-scared.gl.joinmc.link`. Friends connect here — no VPN, no port forwarding required.
- **DuckDNS** (`aarylab.duckdns.org`): tracks the dynamic public IP. Not used for direct inbound access (blocked by CG-NAT), but kept for reference and maintained via a 5-minute cron job.

## Services

| Service | Internal port | External access |
|---------|--------------|----------------|
| Portainer | 9443 | Tailscale only |
| Nginx Proxy Manager | 80 / 443 / 81 | Tailscale only |
| Gitea | 3000 | Tailscale only |
| Woodpecker CI | 8000 | Tailscale only |
| Prometheus | 9090 | Tailscale only |
| Grafana | 3001 | Tailscale only |
| Node Exporter | 9100 | Internal only |
| Calibre-web | — | Tailscale only |
| Minecraft | 25565 | playit.gg → `physical-scared.gl.joinmc.link` |

## Build phases

- [x] **Phase 1 — Security foundation:** UFW default-deny, Fail2ban, SSH key auth (passwords disabled), DuckDNS cron, Tailscale
- [x] **Phase 2 — Container infrastructure:** Docker Engine, Portainer, Nginx Proxy Manager
- [x] **Phase 3 — Dev platform:** Gitea, Woodpecker CI/CD
- [x] **Phase 4 — Observability:** Node Exporter, Prometheus, Grafana (dashboard #1860)
- [ ] **Phase 5 — Documentation:** Architecture diagram, README, LinkedIn posts, resume bullets *(in progress)*

## Why I built this

<!-- Write 2–3 sentences on what drew you to this project and what you wanted to learn. -->

## What I learned

<!-- Write 3–5 paragraphs on key technical concepts, unexpected challenges, and anything that surprised you. -->

## Reproducing this setup

### Prerequisites

- A machine capable of running Proxmox VE (or any Debian 13 VM as a starting point)
- A router with DHCP reservation support
- Free accounts: [DuckDNS](https://www.duckdns.org), [Tailscale](https://tailscale.com), [playit.gg](https://playit.gg) *(game server only)*

### Setup order

```bash
# 1. Install Proxmox VE, create two Debian 13 VMs
#    Set onboot: 1 on both in Proxmox Options

# 2. Phase 1 — security (run on VM 100)
sudo apt install ufw fail2ban -y
sudo ufw default deny incoming && sudo ufw allow 22/tcp && sudo ufw enable
# Configure SSH key auth, disable password login
# DuckDNS cron: */5 * * * * ~/duckdns/duck.sh
curl -fsSL https://tailscale.com/install.sh | sh && sudo tailscale up

# 3. Phase 2 — containers
curl -fsSL https://get.docker.com | sudo sh
# Deploy Portainer and Nginx PM via Docker Compose

# 4. Phase 3 — dev platform
# Deploy Gitea, then Woodpecker; connect via webhook in Gitea settings

# 5. Phase 4 — observability
# Deploy Node Exporter, Prometheus, Grafana
# In Grafana: Connections → Prometheus, then import dashboard ID 1860

# 6. VM 101 — game server
# Install Docker, run itzg/minecraft-server, install playit.gg agent
```

**Critical**: add DHCP MAC reservations in your router for all three devices (Proxmox host + both VMs). IP drift after reboots or network changes will break connectivity.
