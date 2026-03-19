
## Key Rules

- Unit file name **must match the mount path** — slashes become dashes
- Always drop the leading slash
	- `/mnt/data` → `mnt-data.mount`
	- `/mnt/nas/backup` → `mnt-nas-backup.mount`
- Systemd will refuse to load the unit if the name doesn't match the path

---

## Helper Commands

```bash
# Convert a path to the correct unit file name
systemd-escape -p --suffix=mount /mnt/data
# output: mnt-data.mount

# Generate and start a transient mount (no file written, gone on reboot)
sudo systemd-mount /dev/sdb1 /mnt/data

# Generate a persistent unit file written to /etc/systemd/system/
sudo systemd-mount --persistent /dev/sdb1 /mnt/data

# Network share example
sudo systemd-mount --persistent //nas/share /mnt/nas \
  --type=cifs \
  --options=credentials=/etc/samba/creds
```

---

## .mount Unit Template

```ini
# /etc/systemd/system/mnt-data.mount

[Unit]
Description=Mount /mnt/data
# Wait for the block device to be available before mounting
After=blockdev@dev-sdb1.target

[Mount]
# What device/share to mount
What=/dev/sdb1
# Where to mount it
Where=/mnt/data
# Filesystem type
Type=ext4
# Mount options
Options=defaults

[Install]
WantedBy=multi-user.target
```

### NFS Example

```ini
# /etc/systemd/system/mnt-nas.mount

[Unit]
Description=NAS NFS Mount
After=network-online.target
Wants=network-online.target

[Mount]
What=192.168.1.10:/volume1/share
Where=/mnt/nas
Type=nfs
Options=defaults,_netdev

[Install]
WantedBy=multi-user.target
```

> `_netdev` tells systemd this is a network device — ensures it doesn't try to mount before networking is up

---

## .automount Unit Template

Automount triggers the `.mount` only when the path is accessed, then optionally unmounts after idle. Always paired with a matching `.mount` file of the same base name.

```ini
# /etc/systemd/system/mnt-data.automount

[Unit]
Description=Automount /mnt/data

[Automount]
Where=/mnt/data
# Unmount after 5 minutes of inactivity (omit to never auto-unmount)
TimeoutIdleSec=300

[Install]
WantedBy=multi-user.target
```

> Enable the `.automount` not the `.mount` — systemd manages the relationship between them

---

## Managing Mount Units

```bash
# Reload after creating/editing unit files
sudo systemctl daemon-reload

# Mount unit — start/stop/enable
sudo systemctl start mnt-data.mount
sudo systemctl stop mnt-data.mount
sudo systemctl enable mnt-data.mount

# Automount — enable the automount, not the mount
sudo systemctl enable --now mnt-data.automount

# Check status
systemctl status mnt-data.mount
systemctl status mnt-data.automount

# Check all active mounts
systemctl list-units --type=mount

# Check all timers (useful alongside automount for scheduled mounts)
systemctl list-timers --all
```

---

## mount vs automount — When to Use Each

| Scenario                                                 | Use                                   |
| -------------------------------------------------------- | ------------------------------------- |
| Always-on storage (local disk, critical NAS)             | `.mount`                              |
| Occasional network share, don't want it always connected | `.automount`                          |
| Need mount ready before a service starts                 | `.mount` with `After=` in the service |
| Want to save resources, unmount when idle                | `.automount` with `TimeoutIdleSec`    |

## Systemd Mounts/Automounts vs FStab
Fstab has not concept of dependencies, this is where systemd comes in

| Scenario                              | Use                                |
| ----------------------------------------- | ---------------------------------- |
| local disk, simple setup              | fstab                              |
| Network mount, service depends on it  | systemd                            |
| Critical mount, must survive failures | systemd                            |

---

## Tip — Verify Unit Name Before Creating File

Always run `systemd-escape` first to confirm the correct filename before creating the unit:

```bash
systemd-escape -p --suffix=mount /your/mount/path
```

---

## Quick Reference

```bash
systemd-escape -p --suffix=mount /mnt/data   # get correct unit filename
systemd-mount --persistent /dev/sdb1 /mnt/data  # generate unit for you
systemctl daemon-reload                        # reload after changes
systemctl enable --now mnt-data.mount         # enable + start mount
systemctl enable --now mnt-data.automount     # enable + start automount
systemctl list-units --type=mount             # see all active mounts
```