Tags: #linux #ubuntu #networking #systemd #networkd #netplan #sysadmin

---

## Overview

**systemd-networkd** is the network configuration daemon in systemd. It manages interfaces, IP addressing, routing, VLANs, bridges, and bonds using declarative unit files.

**Netplan** is Ubuntu's abstraction layer that sits in front of networkd (or NetworkManager). You write YAML, Netplan translates it to the appropriate backend config files.

```
/etc/netplan/*.yaml        ← Ubuntu Server — write config here
        │
        ▼
Netplan (translator)
        │
        ├──► systemd-networkd   ← default backend on Ubuntu Server
        └──► NetworkManager     ← default backend on Ubuntu Desktop
```

> Netplan is Ubuntu-specific. Other distros (RHEL, Arch, Debian) use their network stack directly with no abstraction layer.

---

## networkd vs NetworkManager

|systemd-networkd|NetworkManager|
|---|---|---|
|Target environment|Servers, containers|Desktops, laptops|
|Config style|Declarative unit files|Dynamic, nmcli/GUI driven|
|WiFi support|Limited|Full|
|Container networking|Strong|Basic|
|Overhead|Minimal|Higher|
|Default Ubuntu Server|Via Netplan|Not default|
|Default Ubuntu Desktop|No|Via Netplan|
|RHEL / Rocky|No|Yes — primary stack|


---

## Key Files

```
/etc/systemd/network/      ← your manual unit files (if writing directly)
/run/systemd/network/      ← runtime files — Netplan writes generated files here
/lib/systemd/network/      ← vendor defaults, never touch
/etc/systemd/resolved.conf ← resolved config, works alongside networkd
```

Config files processed in alphanumeric order — prefix with numbers:

```
10-eth0.network
20-eth1.network
```

### Netplan Generated Files

When you run `netplan apply`, Netplan generates networkd unit files and writes them to `/run/systemd/network/` — not `/etc/systemd/network/`. They are runtime files, regenerated on every `netplan apply`. Never manually edit them.

```bash
# Inspect what Netplan generated without applying
sudo netplan generate
cat /run/systemd/network/10-netplan-eth0.network
```

---

## Config File Types

|Type|Purpose|
|---|---|
|`.network`|Configures an interface — IP, DNS, routes, DHCP|
|`.netdev`|Creates virtual devices — bridges, VLANs, bonds, tunnels|
|`.link`|Low-level interface settings — MAC address, naming, driver options|

---

## .network File Examples

### DHCP

```ini
# /etc/systemd/network/10-eth0.network

[Match]
Name=eth0

[Network]
DHCP=yes
DNS=1.1.1.1
Domains=home.lab
```

### Static IP

```ini
[Match]
Name=eth0

[Network]
Address=192.168.1.50/24
Gateway=192.168.1.1
DNS=192.168.1.1 1.1.1.1
Domains=home.lab
DHCP=no
```

### Match on MAC Instead of Name

Useful when interface names aren't predictable:

```ini
[Match]
MACAddress=00:11:22:33:44:55

[Network]
DHCP=yes
```

---

## .netdev Examples

### VLAN

```ini
# /etc/systemd/network/20-vlan10.netdev
[NetDev]
Name=vlan10
Kind=vlan

[VLAN]
Id=10
```

Attach to physical interface in its `.network` file:

```ini
# /etc/systemd/network/10-eth0.network
[Match]
Name=eth0

[Network]
VLAN=vlan10
```

### Bridge

```ini
# /etc/systemd/network/20-br0.netdev
[NetDev]
Name=br0
Kind=bridge
```

```ini
# /etc/systemd/network/10-eth0.network — attach eth0 as bridge member
[Match]
Name=eth0

[Network]
Bridge=br0
```

```ini
# /etc/systemd/network/30-br0.network — configure the bridge IP
[Match]
Name=br0

[Network]
DHCP=yes
```

### Bond (NIC Teaming)

```ini
# /etc/systemd/network/20-bond0.netdev
[NetDev]
Name=bond0
Kind=bond

[Bond]
Mode=active-backup      # failover mode
MIIMonitorSec=1s        # link check interval
```

```ini
# Attach each physical interface as a bond member
# /etc/systemd/network/10-eth0.network
[Match]
Name=eth0

[Network]
Bond=bond0
```

---

## networkctl — Interface Status

```bash
networkctl
```

```
IDX LINK   TYPE     OPERATIONAL SETUP
  1 lo     loopback carrier     unmanaged
  2 eth0   ether    routable    configured
  3 eth1   ether    degraded    configuring
```

**Operational states:**

|State|Meaning|
|---|---|
|`routable`|Has IP, has default route, fully up|
|`degraded`|Has IP but missing something (route, DNS)|
|`carrier`|Physical link up, no IP yet|
|`off`|Interface down|
|`no-carrier`|No physical link — cable unplugged|

**Setup states:**

|State|Meaning|
|---|---|
|`configured`|networkd applied config successfully|
|`configuring`|In progress|
|`unmanaged`|networkd is not managing this interface|
|`failed`|Config failed to apply|

---

## networkd and resolved Integration

networkd and resolved are designed to work as a pair. networkd pushes DNS info to resolved via DBus automatically — from either DHCP or static config in your `.network` file.

```ini
[Network]
DHCP=yes

[DHCP]
UseDNS=yes       # pass DHCP-provided DNS to resolved
UseRoutes=yes
```

```bash
# Verify resolved received DNS config from networkd
resolvectl status eth0
```

---

## Managing networkd

```bash
# Enable and start
sudo systemctl enable --now systemd-networkd

# Restart after config changes
sudo systemctl restart systemd-networkd

# Reload config without full restart
sudo networkctl reload

# Status
systemctl status systemd-networkd

# Live logs
journalctl -u systemd-networkd -f
```

---

## Netplan Commands (Ubuntu Server)

```bash
# View current config
cat /etc/netplan/*.yaml

# Test with auto-rollback — always use this over apply when remote
sudo netplan try

# Apply permanently
sudo netplan apply

# Generate networkd files without applying
sudo netplan generate

# Debug output
sudo netplan apply --debug
```

> `netplan try` gives you 120 seconds to confirm before auto-rollback. Always use this over `netplan apply` on a remote server — a bad config won't lock you out.

---

## Switching From NetworkManager to networkd (Ubuntu)

Always do this with console access — you will lose connectivity between disabling NM and networkd coming up if anything is wrong.

```bash
sudo systemctl disable --now NetworkManager
sudo systemctl enable --now systemd-networkd
sudo systemctl enable --now systemd-resolved

# Fix resolv.conf symlink
sudo ln -sf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf

# Write your .network config, then restart
sudo systemctl restart systemd-networkd
```

---

## Quick Reference

```bash
networkctl                           # all interfaces and state
networkctl status eth0               # detailed interface info
sudo networkctl reload               # reload config without restart
journalctl -u systemd-networkd -f    # live networkd logs
resolvectl status                    # verify DNS config from networkd
cat /etc/netplan/*.yaml              # view Netplan config (Ubuntu)
sudo netplan try                     # test Netplan config with rollback
sudo netplan apply                   # apply Netplan config
sudo netplan generate                # generate networkd files, inspect only
cat /run/systemd/network/*.network   # view Netplan-generated networkd files
```