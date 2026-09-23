# Remote Access with Tailscale

Local IPs (`192.168.x.x`) only work from devices on the same network. To
reach the stack from anywhere — a different WiFi network, cellular data, a
laptop at work — this project uses [Tailscale](https://tailscale.com/), a
WireGuard-based mesh VPN that gives the host a stable address reachable from
any Tailscale-connected device without opening ports on the router.

It also solves a real problem specific to this stack: **Vaultwarden refuses
to run over plain HTTP.** Browsers only expose the Web Crypto API (which
Vaultwarden needs to encrypt/decrypt vault data) over HTTPS or `localhost`,
so `http://<lan-ip>:8082` throws `Insecure URL not allowed` the moment you
try to create an account. Tailscale's `tailscale serve` feature issues a
real Let's Encrypt certificate for the host's `.ts.net` name and proxies
HTTPS straight to the container, which fixes this cleanly instead of relying
on a browser-flag workaround that only affects one browser on one machine.

## 1. Install Tailscale on the host

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

This prints a login URL — open it, sign in, and approve the device.

## 2. Confirm the host is connected

```bash
tailscale status
```

Should list the host with a `100.x.x.x` Tailscale IP and status
`Connected`.

## 3. Enable HTTPS certificates for the tailnet

In the [Tailscale admin console](https://login.tailscale.com/admin) →
**DNS** → make sure **HTTPS Certificates** is turned on. This has to be
enabled once per tailnet before `tailscale serve` can issue certs.

## 4. Expose each service over HTTPS

```bash
sudo tailscale serve --bg --https=8443 http://localhost:8082   # Vaultwarden
sudo tailscale serve --bg --https=8444 http://localhost:8096   # Jellyfin
sudo tailscale serve --bg --https=8445 http://localhost:3001   # Uptime Kuma
```

Check what's exposed:

```bash
sudo tailscale serve status
```

You'll get URLs like:

```
https://<hostname>.<tailnet-name>.ts.net:8443   → Vaultwarden
https://<hostname>.<tailnet-name>.ts.net:8444   → Jellyfin
https://<hostname>.<tailnet-name>.ts.net:8445   → Uptime Kuma
```

with a genuine, browser-trusted certificate — no warnings, no flags.

## 5. Install Tailscale on client devices

Any device that needs remote access — a second computer, a phone, a family
member's laptop — installs the Tailscale app and signs into the same
account (or accepts an invite from the admin console under **Users** →
**Invite**). Once connected, the `.ts.net` URLs above work from that device
regardless of what network it's on.

## 6. File access over Tailscale (Samba)

The [Samba share](SETUP.md) set up for Jellyfin's media folder is bound to
the host's LAN IP by default, which only works locally. To reach the same
share from a different network, map it using the **Tailscale IP** instead
of the LAN IP:

```
\\<tailscale-ip>\media       (Windows)
smb://<tailscale-ip>/media   (Mac)
```

This works identically whether the client is on the home network or
anywhere else, since it's routed through the Tailscale tunnel rather than
depending on local network reachability.

## Common gotcha: laptop on a different router

If a laptop is connected to a different router/mesh node than the server —
common with mesh WiFi systems where devices can land on either the main
router or a satellite/access point on a different subnet — local IPs won't
resolve even though "everything is connected." Tailscale sidesteps this
entirely: as long as both devices show `Connected` in the Tailscale app,
the `.ts.net` URLs work regardless of which physical router either one is
on.
