# Setup Guide

First-time install on a fresh Zorin OS (Ubuntu-based) machine, from bare OS
to all four services running.

## 1. Install Docker

Ubuntu's default repos ship an outdated `docker-compose` package (or none at
all under the `docker-compose-plugin` name), so install straight from
Docker's own repository:

```bash
sudo apt remove docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc -y

sudo apt update
sudo apt install -y ca-certificates curl gnupg

sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

sudo systemctl enable --now docker
sudo usermod -aG docker $USER
```

Log out and back in (or reboot) so your user can run `docker` without `sudo`.

> If `docker.service` fails to start right after install, a reboot usually
> clears it — see [`TROUBLESHOOTING.md`](TROUBLESHOOTING.md#docker-service-fails-to-start-after-install).

Verify:

```bash
docker run hello-world
```

## 2. Free up port 53 for Pi-hole

Ubuntu's `systemd-resolved` binds port 53 by default, which collides with
Pi-hole's DNS server. Disable its stub listener:

```bash
sudo nano /etc/systemd/resolved.conf
```

Change:
```
#DNSStubListener=yes
```
to:
```
DNSStubListener=no
```

Then fix the resolv.conf symlink and restart:

```bash
sudo rm /etc/resolv.conf
sudo ln -s /run/systemd/resolve/resolv.conf /etc/resolv.conf
sudo reboot
```

After reboot, confirm the port is free:

```bash
sudo lsof -i :53
```
(should return nothing)

See [`TROUBLESHOOTING.md`](TROUBLESHOOTING.md#port-53-already-in-use) if it's
still occupied.

## 3. Clone this repo and configure

```bash
git clone <this-repo-url> ~/homelab
cd ~/homelab
cp .env.example .env
nano .env   # set PIHOLE_WEBPASSWORD, TZ, MEDIA_PATH
```

## 4. Start the stack

```bash
docker compose up -d
docker compose ps
```

All four containers should show `Up`/`Running`. If a container shows a
`Ports` column that's empty when it shouldn't be, see
[`TROUBLESHOOTING.md`](TROUBLESHOOTING.md#container-runs-but-ports-arent-published).

## 5. Open the firewall (if UFW is active)

```bash
sudo ufw status
```

If active, open the ports each service needs:

```bash
sudo ufw allow 8081/tcp comment 'Pi-hole web'
sudo ufw allow 8096/tcp comment 'Jellyfin'
sudo ufw allow 8082/tcp comment 'Vaultwarden'
sudo ufw allow 3001/tcp comment 'Uptime Kuma'
sudo ufw allow 53/tcp comment 'Pi-hole DNS'
sudo ufw allow 53/udp comment 'Pi-hole DNS'
sudo ufw reload
```

## 6. Access each service

Find the host's LAN IP:
```bash
ip a
```
Look for the `inet` address on your active wired/wireless interface (not
`docker0`, `br-...`, or `veth...` — those are internal Docker networking).

| Service | URL |
|---|---|
| Pi-hole | `http://<host-ip>:8081/admin` |
| Jellyfin | `http://<host-ip>:8096` |
| Vaultwarden | `http://<host-ip>:8082` |
| Uptime Kuma | `http://<host-ip>:3001` |

Vaultwarden specifically will refuse to create an account over plain HTTP —
see [`REMOTE-ACCESS.md`](REMOTE-ACCESS.md) to fix that with Tailscale.

## 7. Point your router's DNS at Pi-hole (optional, network-wide ad-blocking)

In your router's DHCP/DNS settings, set the primary DNS server to the host's
LAN IP. **Caution:** if Pi-hole ever goes down while it's the only DNS
server for your whole network, every device loses name resolution. Test on
one device first before committing the whole router to it. See
[`TROUBLESHOOTING.md`](TROUBLESHOOTING.md#router-wide-dns-outage) for what
this looks like when it goes wrong and how to recover.

## Day-2 operations

```bash
# Update all containers
docker compose pull
docker compose up -d

# Stop everything
docker compose down

# Logs for one service
docker logs pihole --tail 50
```

Persistent data lives in `pihole/`, `jellyfin/`, `vaultwarden/`, and
`uptime-kuma/` under the project folder — back these up regularly,
especially `vaultwarden/data`.
