Tags: #linux #ubuntu #dns #systemd #networking #sysadmin

---

## What Is systemd-resolved

A local DNS stub resolver and cache provided by systemd. It sits between applications and upstream DNS servers, handling caching, DNSSEC validation, per-interface DNS routing, and split DNS for VPNs — without any manual scripting.

---

## Stub vs Full Recursive DNS

This is the most important concept to understand about resolved.

**Stub resolver (resolved)** — does not do the full DNS walk itself. It forwards requests to an upstream DNS server, caches responses, and handles DNSSEC validation. It trusts the upstream to do the heavy lifting.

**Full recursive resolver (unbound, BIND)** — does the entire DNS walk itself with no upstream needed:

```
Ask root (.)              → "Who knows about .com?"
Ask .com servers          → "Who knows about example.com?"
Ask example.com servers   → "Here is the IP for www.example.com"
```

resolved is intentionally a stub — lightweight, fast, integrated with the system. If you need full recursive resolution (privacy, no upstream trust) you'd run unbound alongside or instead.

---

## How It Works

### The Request Flow

```
Application
    │
    ▼
glibc / system resolver
    │
    ▼
systemd-resolved (stub at 127.0.0.53)
    │  checks cache first
    │  applies per-interface DNS routing
    │  validates DNSSEC if enabled
    ▼
Upstream DNS
(ISP, internal server, VPN DNS, etc.)
```

### /etc/resolv.conf

On a resolved system this file is a symlink, not a real file:

```bash
ls -la /etc/resolv.conf
# /etc/resolv.conf -> ../run/systemd/resolve/stub-resolv.conf

cat /etc/resolv.conf
# nameserver 127.0.0.53
```

Apps read `/etc/resolv.conf`, see `127.0.0.53`, and send all DNS queries there. They have no knowledge of where DNS actually goes — resolved owns that logic entirely.

> The old mental model: `/etc/resolv.conf` = truth 
> The resolved mental model: 
> 	`/etc/resolv.conf` = pointer
> 	resolved = logic
> 	upstream = brains

### DBus Integration

Resolved uses DBus to gather DNS configuration from NetworkManager, systemd-networkd, and VPN clients. This is how it reacts automatically to:

- DHCP pushing new DNS servers
- VPN coming up with its own DNS
- Network interface changes
- Split DNS config changes

No scripts or manual restarts needed — resolved updates its routing table dynamically.

---

## Key Features

**DNS caching** — responses are cached in memory, reducing latency and upstream query volume. Cache is flushed on network changes.

**Per-interface DNS routing** — different DNS servers per interface. The DNS server queried is chosen based on the domain being resolved, not a single global setting. Critical for VPN split DNS.

**DNSSEC validation** — cryptographic verification that DNS responses haven't been tampered with. Optional, configurable per deployment.

**LLMNR / mDNS** — resolves `.local` hostnames on the local network without a DNS server. Useful in homelabs for machine-to-machine discovery.

**DNS over TLS (DoT)** — encrypts DNS queries to upstream server. Prevents ISP snooping on DNS traffic.

**Immutable OS compatibility** — DNS config is declarative and managed by systemd, so it works cleanly on systems where `/etc` may be read-only.

---

## Split DNS — How It Works

The most practical feature for VPN and multi-network environments.

Each interface can have its own DNS server and search domains registered with resolved. When a query comes in, resolved checks which interface's DNS server should handle it based on the domain:

```
Query: server01.company.internal
  → matches VPN interface domain "company.internal"
  → sent to VPN DNS server 10.10.0.1

Query: google.com
  → no specific match
  → sent to default upstream DNS (your router/ISP)
```

**NetworkManager and systemd-networkd act as the bridge**. When a VPN client (like OpenVPN, WireGuard, or F5 BIG-IP) brings up a VPN interface, it tells NetworkManager or systemd-networkd "here are the DNS servers and domains for this interface." Those network managers then pass that information to resolved via DBus. Resolved updates its routing table and starts sending matching queries to the VPN's DNS server.

---

## Configuration

Main config file: `/etc/systemd/resolved.conf`

```ini
[Resolve]
# Fallback upstream DNS if interface provides none
DNS=1.1.1.1 8.8.8.8

# Used if primary DNS fails entirely
FallbackDNS=9.9.9.9

# Default search domain
Domains=home.lab

# DNSSEC — yes (strict), allow-downgrade (best effort), no (off)
DNSSEC=allow-downgrade

# DNS over TLS — yes (strict), opportunistic (best effort), no (off)
DNSOverTLS=opportunistic

# Cache negative responses (NXDOMAIN)
Cache=yes
```

After changes:

```bash
sudo systemctl restart systemd-resolved
```

---

## Key Commands

```bash
# Full status — interfaces, DNS servers, DNSSEC, feature flags
resolvectl status

# DNS lookup through resolved
resolvectl query google.com

# Which DNS server answered a query
resolvectl query --interface=eth0 google.com

# Flush DNS cache
resolvectl flush-caches

# Cache and query statistics
resolvectl statistics

# Monitor DNS queries in real time (great for troubleshooting)
resolvectl monitor

# Check active DNS servers per interface
resolvectl dns

# Check DNSSEC status
resolvectl dnssec

# Service logs
journalctl -u systemd-resolved -f
```

---

## Troubleshooting

```bash
# Confirm resolved is handling DNS
cat /etc/resolv.conf
# should show: nameserver 127.0.0.53

# Test resolution and see which server answered
resolvectl query github.com

# Bypass resolved, query upstream directly (compare results)
dig @1.1.1.1 github.com

# Check if a specific interface has DNS assigned
resolvectl status eth0

# Flush cache after config changes
resolvectl flush-caches

# VPN DNS not working — check if VPN registered its DNS with resolved
resolvectl status   # look for VPN interface entry
```

Common issue — VPN DNS not resolving while general DNS works fine. Almost always means the VPN client didn't register its DNS servers with resolved properly. Check `resolvectl status` and look for the VPN interface — if DNS is missing there, the VPN client config needs fixing, not resolved.

---

## Containers and resolved

Containers (Docker,K8s) don't use the host's resolved instance. The loopback address `127.0.0.53` is host-namespaced — containers can't reach it. Instead containers get their own simple DNS config, usually:

- Docker injects its own internal DNS at `127.0.0.11`
- Or the container is handed the host's upstream DNS IP directly
- Kubernetes uses CoreDNS 

---

## resolved vs Alternatives

|Tool|Type|Use case|
|---|---|---|
|systemd-resolved|Stub|Default system resolver, per-interface DNS, VPN split DNS|
|unbound|Full recursive|Privacy, no upstream trust, internal DNS server|
|dnsmasq|Stub + DHCP|Lightweight, common on routers and homelab DNS/DHCP combos|
|BIND9|Full authoritative + recursive|Enterprise, AD environments, full zone management|

For a homelab the common pattern is resolved on each machine handling local resolution, with unbound or dnsmasq running on a dedicated DNS server for the whole network.

---

## Quick Reference

```bash
resolvectl status                  # full DNS status per interface
resolvectl query <domain>          # test DNS lookup
resolvectl monitor                 # watch DNS queries live
resolvectl flush-caches            # flush DNS cache
resolvectl statistics              # cache hit/miss stats
cat /etc/resolv.conf               # confirm pointing at 127.0.0.53
journalctl -u systemd-resolved -f  # live logs
dig @1.1.1.1 <domain>             # bypass resolved, test upstream directly
```