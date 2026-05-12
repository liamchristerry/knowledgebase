Tags: #linux #ubuntu #systemd #sysadmin #time #encryption #kernal 

---

## Targets

Targets are groupings of units that represent a system state. They replaced SysV runlevels (0-6) — instead of a single number representing the system state, targets are named and can have dependencies on each other.

Targets don't do work themselves — they pull in other units that do.

```
graphical.target
  ├─ multi-user.target
  │   ├─ sshd.service
  │   ├─ network.target
  │   └─ crond.service
  └─ gdm.service
```

### Common Targets

|Target|Meaning|
|---|---|
|`multi-user.target`|Normal non-GUI server — equivalent to SysV runlevel 3|
|`graphical.target`|GUI desktop — equivalent to SysV runlevel 5|
|`rescue.target`|Single-user rescue mode, minimal services, root only|
|`emergency.target`|Most minimal — root filesystem read-only, almost nothing loaded|
|`reboot.target`|Triggers clean shutdown and reboot|
|`poweroff.target`|Triggers clean shutdown and power off|
|`network.target`|Network interfaces are up (not necessarily online)|
|`network-online.target`|Network is fully up and routable — use this in After=|

> `network.target` vs `network-online.target` is a common gotcha — services that need actual network connectivity should use `network-online.target` in their `After=` and `Wants=` directives, not just `network.target`.

### Commands

```bash
# Show current default target (what boots into by default)
systemctl get-default

# Change default target
sudo systemctl set-default multi-user.target

# Switch to a target immediately (affects running system)
sudo systemctl isolate rescue.target

# List all available targets
systemctl list-units --type=target
```

> `isolate` immediately stops all units not pulled in by the target and starts those that are. Useful for dropping into rescue mode without rebooting.

---

## timesyncd

A lightweight SNTP/NTP client built into systemd. Handles time synchronisation for most systems without needing a full NTP daemon.

### timesyncd vs chronyd vs ntpd

| timesyncd             | chronyd               | ntpd                          |          |
| --------------------- | --------------------- | ----------------------------- | -------- |
| Type                  | SNTP client           | Full NTP                      | Full NTP |
| Serves time to others | No                    | Yes                           | Yes      |
| Peer syncing          | No                    | Yes                           | Yes      |
| Advanced algorithms   | No                    | Yes                           | Yes      |
| Resource usage        | Minimal               | Low                           | Low      |
| Use case              | Clients, VMs, servers | Servers, accurate timekeeping | Legacy   |

**Use timesyncd when** the machine just needs to sync its own clock. **Use chronyd when** you need to serve NTP to other machines, need high accuracy, or are running in an environment with intermittent network (chronyd handles clock drift better).

### How It Works

- Contacts configured NTP servers
- Steps the clock early in boot if offset is large (big jump to correct time)
- Slews the clock after boot (gradually adjusts rather than jumping — prevents log timestamp issues)
- Does not serve time to other machines

### Config — `/etc/systemd/timesyncd.conf`

```ini
[Time]
NTP=pool.ntp.org
FallbackNTP=ntp.ubuntu.com
RootDistanceMaxSec=5
PollIntervalMinSec=32
PollIntervalMaxSec=2048
```

### Commands

```bash
# Full time sync status
timedatectl

# Detailed sync status — server, offset, delay, poll interval
timedatectl timesync-status

# Show NTP servers in use
timedatectl show-timesync

# Enable NTP sync
sudo timedatectl set-ntp true

# Service status
systemctl status systemd-timesyncd
```

> Worth going deeper on: clock stepping vs slewing behaviour matters in environments with strict log correlation (SIEM, audit logs). A clock step can cause log timestamps to jump, which breaks event ordering in security tools.

---

## systemd-cryptsetup

systemd's integration layer for LUKS disk encryption. Handles unlocking encrypted volumes during boot before filesystems are mounted.

### Boot Flow

```
initramfs
    │
    ▼
systemd-cryptsetup     ← reads /etc/crypttab
    │                    creates transient units per encrypted volume
    ▼                    e.g. systemd-cryptsetup@root.service
Decrypt volume
    │
    ▼
systemd-fsck           ← check filesystem integrity
    │
    ▼
Mount filesystem
    │
    ▼
Normal boot continues
```

### /etc/crypttab

Defines encrypted volumes and how to unlock them:

```bash
# name          device                  keyfile     options
cryptroot       UUID=xxxx-xxxx          none        luks
cryptdata       /dev/sdb1               /etc/keys/data.key  luks
```

```bash
# Manually unlock a volume (useful in rescue situations)
sudo cryptsetup luksOpen /dev/sdb1 cryptdata

# Check LUKS volume info
sudo cryptsetup luksDump /dev/sdb1

# List open encrypted volumes
ls /dev/mapper/
```

> Worth going deeper on: LUKS keyfiles, TPM-based auto-unlock (systemd-cryptenroll on modern systems), and what happens when cryptsetup fails at boot — knowing how to recover from a failed unlock is important.

---

## systemd-fsck

Filesystem integrity checking — ensures filesystems aren't corrupt before mounting. Especially important after crashes or unclean shutdowns.

### How It Works

- Triggered automatically per filesystem at boot
- Uses the appropriate tool per filesystem type:
    - ext4 → `fsck.ext4`
    - xfs → `xfs_repair`
    - btrfs → `btrfs check`
- Root filesystem checked early and in isolation
- Non-root filesystems can run in parallel — speeds up boot on systems with many mounts

Units look like:

```
systemd-fsck@dev-disk-by\x2duuid-XXXX.service
```

### Commands

```bash
# Manually run fsck (filesystem must be unmounted)
sudo fsck /dev/sdb1

# Force check on next boot (ext4)
sudo tune2fs -C 100 /dev/sda1    # fake high mount count triggers check

# View fsck results from last boot
journalctl -u systemd-fsck* -b
```

> ZFS and Btrfs handle their own integrity checking — systemd-fsck doesn't apply to them. ZFS scrub and Btrfs scrub are the equivalents and should be run on a schedule via a systemd timer.

---

## logind

Manages user logins and sessions. Tracks who is logged in, from where, and what they have access to. Also handles power actions.

### What It Tracks

- Active users and sessions
- TTY and PTY assignments
- Seat assignments (physical console + input/output devices)
- Idle detection
- Power button / lid actions (primarily desktop relevant)

### Commands

```bash
# List all active sessions
loginctl list-sessions

# Detailed info on a session
loginctl show-session <ID>

# List logged in users
loginctl list-users

# Terminate a session
loginctl terminate-session <ID>

# Lock a session
loginctl lock-session <ID>
```

```bash
# Relevant config
/etc/systemd/logind.conf

# Common tunable — how long before idle session is killed
IdleAction=poweroff
IdleActionSec=30min

# Allow non-root users to shutdown/reboot
HandlePowerKey=poweroff
```

---

## sysctl (systemd-sysctl)

Applies kernel tuning parameters early in boot from config files. Ensures tunings are declarative and consistent rather than applied via boot scripts.

### Config File Locations (in order of precedence)

```
/etc/sysctl.d/*.conf          ← your custom settings
/run/sysctl.d/*.conf          ← runtime settings
/usr/lib/sysctl.d/*.conf      ← vendor defaults
/etc/sysctl.conf              ← legacy catch-all
```

### Common Settings

```bash
# Enable IP forwarding (required for routing, VMs, containers)
net.ipv4.ip_forward = 1
net.ipv6.conf.all.forwarding = 1

# Reduce swap aggressiveness (good for servers, databases)
vm.swappiness = 10

# Increase inotify watches (needed for large codebases, Dropbox, Syncthing)
fs.inotify.max_user_watches = 524288

# Increase network buffer sizes (high throughput servers)
net.core.rmem_max = 134217728
net.core.wmem_max = 134217728

# Disable IPv6 if not needed
net.ipv6.conf.all.disable_ipv6 = 1
```

### Commands

```bash
# Apply all sysctl config files without rebooting
sudo sysctl --system

# Apply a single file
sudo sysctl -p /etc/sysctl.d/99-custom.conf

# View a specific setting
sysctl net.ipv4.ip_forward

# Set a value temporarily (lost on reboot)
sudo sysctl -w net.ipv4.ip_forward=1

# View all current settings
sysctl -a
```

> `net.ipv4.ip_forward = 1` is required on any Linux host acting as a router or running containers/VMs that need network access — relevant to your Proxmox and Docker hosts.

---

## systemd-modules-load

Loads kernel modules during boot from config files. Replaces ad-hoc modprobe calls in boot scripts.

### Config Locations

```
/etc/modules-load.d/*.conf
/usr/lib/modules-load.d/*.conf
```

### Example Config

```bash
# /etc/modules-load.d/networking.conf

br_netfilter      # required for Kubernetes/container bridge firewalling
overlay           # overlay filesystem for containers
nf_conntrack      # connection tracking for iptables/nftables
8021q             # VLAN support
bonding           # NIC bonding
```

### Commands

```bash
# List currently loaded modules
lsmod

# Load a module immediately
sudo modprobe br_netfilter

# Remove a module
sudo modprobe -r br_netfilter

# Get info on a module
modinfo br_netfilter

# Check if a module loaded at boot
journalctl -u systemd-modules-load -b
```

> `br_netfilter` and `overlay` are required before deploying Kubernetes or k3s — they're typically added to modules-load.d as part of node prep.

---

## systemd-udevd

The device manager for Linux. Handles hardware discovery, device naming, permissions, and hotplug events. No udevd means no `/dev` nodes — no disks, NICs, or GPUs.

### How It Works

```
Kernel detects hardware → sends uevent
        │
        ▼
systemd-udevd receives uevent
        │
        ├── applies udev rules
        ├── creates /dev/sdb node
        ├── sets permissions and ownership
        ├── creates symlinks (/dev/disk/by-uuid/, /dev/disk/by-id/)
        └── triggers dependent systemd units
```

### Rule Locations

```
/etc/udev/rules.d/         ← your custom rules (take priority)
/usr/lib/udev/rules.d/     ← vendor/package rules
```

### Common Use Cases for Custom Rules

```bash
# /etc/udev/rules.d/99-custom.rules

# Assign a persistent name to a NIC by MAC address
SUBSYSTEM=="net", ACTION=="add", ATTR{address}=="00:11:22:33:44:55", NAME="wan0"

# Set permissions on a USB device
SUBSYSTEM=="usb", ATTR{idVendor}=="1234", MODE="0666"

# Run a script when a device is plugged in
SUBSYSTEM=="block", ACTION=="add", RUN+="/usr/local/bin/handle-disk.sh"
```

### Commands

```bash
# Reload udev rules without rebooting
sudo udevadm control --reload-rules
sudo udevadm trigger

# Test what rules would apply to a device
udevadm test /sys/block/sdb

# Monitor udev events in real time (useful when plugging in hardware)
udevadm monitor

# Get device attributes (used to write rules)
udevadm info /dev/sdb
udevadm info --attribute-walk /dev/sdb
```

> `udevadm monitor` is genuinely useful for troubleshooting — run it before plugging in a device and you'll see exactly what the kernel reports and what udev does with it.

---

## systemd-nspawn

Lightweight container runtime built into systemd. Uses Linux namespaces and cgroups. Think of it as chroot with proper isolation — not a replacement for Docker or Kubernetes but useful for specific tasks.

### When to Use It

- Booting a minimal OS image for testing
- Building packages in an isolated environment
- Testing systemd-based workflows without a full VM
- Quick throwaway environments

### Commands

```bash
# Boot a container from a directory
sudo systemd-nspawn -D /path/to/rootfs -b

# Run a single command in a container
sudo systemd-nspawn -D /path/to/rootfs /bin/bash

# Boot with network access
sudo systemd-nspawn -D /path/to/rootfs --network-veth -b
```

---

## systemd-machined

Tracks running machines — VMs, containers, and nspawn instances. Provides a unified inventory of what's running on the host regardless of how it was started.

```bash
# List all running machines
machinectl list

# Get a shell inside a running machine
machinectl shell <name>

# Show details of a machine
machinectl status <name>

# Stop a machine
machinectl stop <name>
```

---

## Quick Reference — Things Worth Going Deeper On

These topics came up in notes but warrant their own deeper dive when relevant:

- **LUKS / cryptsetup** — TPM-based auto-unlock, keyfile management, recovery from failed unlock
- **chronyd vs timesyncd** — if any of your servers need to serve NTP internally, chronyd is the right tool
- **udev rules** — persistent NIC naming and disk identification by UUID vs by-id vs by-path matters for Proxmox and storage builds
- **sysctl tuning** — specific profiles for Proxmox hosts, Docker hosts, and database servers differ significantly
- **network-online.target** — understanding exactly when this fires and its limitations is important for services that depend on network being truly ready