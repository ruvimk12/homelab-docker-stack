# Adding Storage

How to add a second drive to the host and put it to work — more media
storage, backups, a fully separate lab environment, or an additional network
share.

## 1. Add and mount the drive

```bash
lsblk                                  # identify the new drive, e.g. /dev/sdb1

sudo mkfs.ext4 /dev/sdb1               # ONLY if the drive is new/empty — this wipes it

sudo mkdir -p /mnt/storage2
sudo mount /dev/sdb1 /mnt/storage2
```

Make it mount automatically on boot:

```bash
sudo blkid /dev/sdb1                   # copy the UUID
sudo nano /etc/fstab
```

Add:
```
UUID=<your-uuid-here>  /mnt/storage2  ext4  defaults  0  2
```

Test it:
```bash
sudo mount -a
sudo chown -R $USER:$USER /mnt/storage2
```

## 2. What to build on it

| Goal | What it's for |
|---|---|
| More Jellyfin storage | Expand media space beyond the main drive |
| Backup drive | Separate copy of Vaultwarden/Pi-hole/photo data |
| Second Docker stack | An isolated lab environment kept apart from daily-use services |
| Expanded Samba share | A new network drive visible from Windows/Mac |

### Option A — More Jellyfin storage

Bind-mount the new drive into the Jellyfin container by adding a line under
its `volumes:` in `docker-compose.yml`:

```yaml
    volumes:
      - /mnt/storage2:/storage2
```

Then `docker compose up -d` to apply it, and add `/storage2` as a folder in
a new or existing Jellyfin library.

### Option B — Backup drive

One-off copy:
```bash
rsync -av ~/homelab/ /mnt/storage2/backup/
```

Automate nightly with cron:
```bash
crontab -e
```
```
0 2 * * *  rsync -av /home/<user>/homelab/ /mnt/storage2/backup/
```

### Option C — Second, separate Docker stack

Keep an experimental or lab environment (e.g. a Windows Server/AD test lab)
fully isolated from the daily-use stack:

```bash
mkdir -p /mnt/storage2/lab2
cd /mnt/storage2/lab2
# write a separate docker-compose.yml here
docker compose up -d
```

### Option D — Expand the Samba share

Add a new share block in `/etc/samba/smb.conf`:

```
[storage2]
   path = /mnt/storage2
   browseable = yes
   read only = no
   guest ok = no
   valid users = <your-username>
```

```bash
sudo systemctl restart smbd
```

Map it from Windows the same way as the media share, using either the LAN
IP or the Tailscale IP (see [`REMOTE-ACCESS.md`](REMOTE-ACCESS.md)):

```
\\<lan-ip>\storage2         (local network)
\\<tailscale-ip>\storage2   (from anywhere)
```
