# Troubleshooting Log

Real issues hit while standing up this stack, kept here rather than edited
away — the diagnosis process is the point.

## Docker service fails to start after install

**Symptom:**
```
Job for docker.service failed because the control process exited with error code.
See "systemctl status docker.service" and "journalctl -xeu docker.service" for details.
```
right after installing `docker-ce` from the official repo.

**Diagnosis:** `systemctl status containerd.service` showed containerd
itself running fine, which narrowed it to docker's own service rather than
its dependency. On a fresh install (or right after a kernel update), a
reboot is often enough to clear a stale service-manager state that a
same-session `systemctl start` can't fix.

**Fix:**
```bash
sudo reboot
```
After reboot: `sudo systemctl status docker` showed `active (running)`.
**Lesson:** check the dependency (`containerd`) separately from the service
that's actually failing (`docker`) before assuming the wrong component is
broken.

---

## Port 53 already in use

**Symptom:** Pi-hole's container fails to bind:
```
Error response from daemon: failed to set up container networking: driver failed programming
external connectivity on endpoint pihole: failed to bind host port 0.0.0.0:53:tcp: address already in use
```

**Diagnosis:**
```bash
sudo lsof -i :53
```
showed `systemd-resolved` already bound to port 53 — Ubuntu's built-in DNS
stub resolver runs by default and conflicts with any container that also
wants port 53.

**Fix:** disable `systemd-resolved`'s stub listener (`DNSStubListener=no`
in `/etc/systemd/resolved.conf`), fix `/etc/resolv.conf`'s symlink, and
reboot. A `systemctl restart systemd-resolved` alone was not sufficient in
practice — the change only took effect reliably after a full reboot.
**Lesson:** verify a config change actually landed (`grep` the setting back
out of the file) before assuming a restart applied it — the first two
restart attempts silently kept the old value because the edit hadn't been
saved before the file was viewed again.

---

## Container runs but ports aren't published

**Symptom:** `docker compose ps` showed Pi-hole as `Up ... (healthy)` but
with an **empty** `PORTS` column, while every other container showed its
mapping correctly. `curl http://localhost:8081/admin` failed even
though the container was "running."

**Diagnosis:** the container had originally been created during a failed
`docker compose up` (the run that hit the port-53 conflict above). Later
successful runs just restarted that same container object instead of
recreating it, so it never picked up a working port mapping even after the
underlying conflict was fixed.

**Fix:** force recreation instead of a plain restart:
```bash
docker compose stop pihole
docker compose rm -f pihole
docker compose up -d pihole
```
**Lesson:** `docker compose up -d` does not always recreate a container just
because its dependencies changed — a container born from a failed run can
persist its broken state indefinitely until explicitly removed.

---

## Pi-hole password doesn't match `.env`

**Symptom:** after the container-recreation fix above, the password set in
`WEBPASSWORD` no longer logged in.

**Diagnosis:** Pi-hole persists its admin password inside the
`/etc/pihole` volume on first run, not just from the environment variable.
Since the volume survived across the failed attempts, it still held
whatever password had been set the first time the container ever
initialized — the env var only applies on a truly fresh volume.

**Fix:** set it directly against the running container instead of relying
on the environment variable:
```bash
docker exec -it pihole pihole setpassword
```
**Lesson:** for services with persistent config volumes, an env var is only
authoritative on first initialization — a stale volume silently overrides
it on every subsequent start.

---

## Vaultwarden refuses to load over plain HTTP

**Symptom:**
```
You are not using a secure context which is required for the Subtle Crypto API to work.
You need to enable HTTPS!
```
and, when forcing past it with a browser flag, account creation still fails
with `Insecure URL not allowed. All URLs must use HTTPS.`

**Diagnosis:** this isn't a bug in the setup — browsers only expose the Web
Crypto API over HTTPS or `localhost`, and Vaultwarden's own server-side code
independently re-checks for HTTPS on account creation regardless of what
the browser allows. A `chrome://flags` / `brave://flags` workaround
(`unsafely-treat-insecure-origin-as-secure`) fixed the browser-side check
but not Vaultwarden's own server-side check — so it got further but still
failed.

**Fix:** proper HTTPS via Tailscale's `tailscale serve`, which issues a real
certificate for the host's `.ts.net` name (see
[`REMOTE-ACCESS.md`](REMOTE-ACCESS.md)) — not a workaround, the actual fix,
and it also solves remote access at the same time.
**Lesson:** a client-side workaround (browser flag) can mask one half of a
two-sided check and create a false sense of progress; when an error
persists after the "fix," check whether the other side (server-side
validation) has the same requirement independently.

---

## Router-wide DNS outage

**Symptom:** after pointing the router's primary DNS at the Pi-hole host
(so ad-blocking applies network-wide), the entire network — every device,
not just one — lost internet access, and the mesh router's satellite and
main unit both showed an amber/orange status light.

**Diagnosis:** some mesh router firmware uses DNS resolution to validate its
own upstream internet connectivity. Once every device (including the router
itself) pointed at a single Pi-hole instance and something in that chain
hiccuped, the router concluded it had no internet at all, not just that DNS
was misbehaving — cascading a single point of failure across the whole
network rather than just breaking ad-blocking.

**Fix, in order attempted:**
1. Reboot the satellite unit — no effect (the DNS setting lives on the main
   router, not the satellite).
2. Reboot the main router — resolved a lesser version of this same failure
   earlier, but by this point login credentials for the admin panel had
   also been forgotten, blocking a clean DNS-setting rollback.
3. Factory reset the main router (recovery-question flow was a dead end
   with no way to bypass it) — restored default DNS (automatic/ISP) and
   full connectivity, at the cost of re-pairing the mesh satellite and
   reconnecting every device to a new WiFi network.

**Lesson:** router-wide DNS should be tested on **one device first**
(override that device's DNS manually) before committing the entire
network's DHCP to a single self-hosted resolver — a home lab service going
down should degrade gracefully, not take the whole house offline. Also:
record router admin credentials somewhere durable (e.g. in Vaultwarden,
once it's up) before they're needed for recovery.

---

## SMB share: "Destination Folder Access Denied" from Windows

**Symptom:** the Samba share connected successfully from Windows (folder
listing visible), but copying files into it failed with a Windows
permissions dialog.

**Diagnosis:** the media folder had been created by Docker (as `root`) when
Jellyfin first initialized its library, so the Samba user account — a
normal, non-root Linux user — had no write permission into it, even though
the SMB *authentication* succeeded.

**Fix:**
```bash
sudo chown -R $USER:$USER ~/homelab/media
sudo chmod -R 775 ~/homelab/media
```
**Lesson:** a successful network-share login only proves authentication
works — it says nothing about filesystem-level permissions underneath,
which is a separate layer that has to be checked independently.

---

## SMB share disappears on a different network

**Symptom:** `\\192.168.1.x\media` worked fine at home, but silently failed
to reconnect ("Attempting to connect..." → timeout) from the same laptop on
a different network.

**Diagnosis:** not a bug — a LAN IP is only ever routable from devices on
that same local network. Switching networks makes the address
unreachable by definition, regardless of Samba, firewall, or credential
configuration.

**Fix:** map the same share using the host's **Tailscale IP** instead of its
LAN IP (`\\100.x.x.x\media`), which resolves correctly regardless of which
physical network the client is on. See
[`REMOTE-ACCESS.md`](REMOTE-ACCESS.md).
