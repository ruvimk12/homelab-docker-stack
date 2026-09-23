# Home Lab Server

![Docker](https://img.shields.io/badge/Docker-Compose-2496ED?logo=docker&logoColor=white)
![Linux](https://img.shields.io/badge/Zorin_OS-Ubuntu--based-89C441?logo=linux&logoColor=white)
![Tailscale](https://img.shields.io/badge/Tailscale-mesh_VPN-242424?logo=tailscale&logoColor=white)
![License](https://img.shields.io/badge/license-MIT-informational)

A self-hosted services stack running on Zorin OS, built to be used every
day — not spun up once for a lab exercise and torn down. Four production
services run in Docker, reachable securely from any network via a
Tailscale-backed WireGuard mesh with real HTTPS, and every failure hit along
the way is documented with root cause and fix rather than edited out.

**Why it's here:** most home-lab repos show a clean final `docker-compose.yml`
and call it done. The setup process is where the actual troubleshooting
happens — a DNS port conflict, a container that silently kept a stale
network config after a failed run, a two-sided HTTPS requirement that a
client-side workaround only half-fixed, a mesh router that took the whole
house offline when DNS was pointed at a single host. Each of those is
written up in [`TROUBLESHOOTING.md`](TROUBLESHOOTING.md) as
symptom → diagnosis → fix → lesson, because that's the part of the work
worth showing.

## Skills demonstrated

- **Linux system administration** — service management (systemd), DNS
  resolution internals, disk/volume management, firewall configuration (UFW)
- **Containerization** — multi-service orchestration with Docker Compose,
  volume persistence, networking, debugging stale container state
- **Networking** — DNS-level ad blocking, mesh/DHCP troubleshooting,
  double-NAT diagnosis, VPN mesh (WireGuard via Tailscale), TLS/HTTPS
  certificate provisioning
- **File services** — Samba/SMB share configuration and cross-platform
  Linux/Windows permissions troubleshooting
- **Documentation** — root-cause writeups intended to be read by someone
  else, not just notes to self

## Stack

| Service | Purpose | Port |
|---|---|---|
| [Pi-hole](https://pi-hole.net/) | Network-wide DNS ad-blocking | 8081 |
| [Jellyfin](https://jellyfin.org/) | Media server (photos/videos) | 8096 |
| [Vaultwarden](https://github.com/dani-garcia/vaultwarden) | Self-hosted password manager (Bitwarden-compatible) | 8082 |
| [Uptime Kuma](https://github.com/louislam/uptime-kuma) | Status dashboard / monitoring | 3001 |

All four run as containers defined in a single [`docker-compose.yml`](docker-compose.yml).

## Architecture

```
                       ┌─────────────────────────────┐
                       │   Zorin OS host (Docker)     │
                       │                              │
  LAN clients ───────▶ │  Pi-hole · Jellyfin          │
  (192.168.x.x)        │  Vaultwarden · Uptime Kuma   │
                       │                              │
  Remote clients ─────▶│  same containers, reached    │
  (Tailscale tailnet)  │  via Tailscale + HTTPS        │
                       └─────────────────────────────┘
```

- **Local access** — any device on the same LAN hits the host's local IP
  directly on each service's port.
- **Remote access** — [Tailscale](https://tailscale.com/) gives the host a
  stable identity reachable from any network without port-forwarding, and
  `tailscale serve` terminates real HTTPS certificates in front of each
  service (needed for Vaultwarden specifically — browsers refuse its crypto
  APIs over plain HTTP).
- **File access** — a Samba (SMB) share exposes the Jellyfin media folder to
  Windows/Mac clients for drag-and-drop uploads, reachable over either the
  LAN IP or the Tailscale IP.

## Repo layout

```
.
├── README.md                   this file
├── docker-compose.yml          the full stack definition
├── .env.example                template for secrets/config (copy to .env)
├── docs/
│   ├── SETUP.md                 first-time install, start to finish
│   ├── ADDING-STORAGE.md         mounting a new drive & what to build on it
│   ├── REMOTE-ACCESS.md          Tailscale + HTTPS + Samba over Tailscale
│   └── TROUBLESHOOTING.md        real issues hit during setup, and the fix
└── LICENSE
```

## Quick start

```bash
git clone https://github.com/<your-username>/homelab-docker-stack.git
cd homelab-docker-stack
cp .env.example .env      # then edit .env with your own values
docker compose up -d
```

Full walkthrough, including OS-level prerequisites (Docker install, DNS port
conflict, firewall) in [SETUP.md](SETUP.md).

## Notes on this being public

- No real passwords, IPs, or Tailscale hostnames are committed — everything
  sensitive is pulled from a `.env` file that is git-ignored (see
  `.env.example` for the shape of it).
- The addresses that appear in the docs (`192.168.1.x`, `*.ts.net`) are
  placeholders standing in for values that are specific to this deployment.

## License

MIT — see [`LICENSE`](LICENSE).
