
Tags: #proxmox #networking #linux #sysadmin #homelab #vlan #bridge

---
## Referances
https://docs.bankai-tech.com/Proxmox/Docs/Networking/Network%20Configuration/#advanced-bridge-configurations
## Network Architecture Overview

Proxmox networking is built on Linux bridges. Physical NICs attach to bridges, and VMs/LXC containers connect to those bridges — giving them access to the physical network.

```
┌─────────────────────────────────────────┐
│              Proxmox Host               │
│                                         │
│  ┌──────┐   ┌──────┐   ┌──────┐         │
│  │  VM1 │   │  VM2 │   │  LXC │         │
│  └──┬───┘   └──┬───┘   └──┬───┘         │
│     └──────────┼───────────┘            │
│                │                        │
│        ┌───────┴───────┐                │
│        │ Linux Bridge  │                │
│        │   (vmbr0)     │                │
│        └───────┬───────┘                │
│                │                        │
│        ┌───────┴───────┐                │
│        │ Physical NIC  │                │
│        │   (enp1s0)    │                │
│        └───────────────┘                │
└─────────────────────────────────────────┘
```

---

## Recommended Network Separation

The most important design decision in Proxmox networking — separate traffic types onto dedicated bridges. Mixing management, VM, and storage traffic on one interface means a saturated VM network can take down host access or degrade storage performance.

```
Physical NICs
├── enp1s0 + enp2s0 → bond0 → vmbr0  (management + VM trunk, VLAN-aware)
├── enp3s0          →         vmbr1  (storage — iSCSI, NFS, Ceph)
└── enp4s0          →         vmbr2  (corosync cluster communication)
```

|Bridge|Purpose|Notes|
|---|---|---|
|`vmbr0`|Management + VM traffic|VLAN-aware, Proxmox UI and SSH here|
|`vmbr1`|Storage traffic|Dedicated NIC, jumbo frames, no gateway|
|`vmbr2`|Cluster/corosync|Dedicated NIC, cluster stability depends on this|

> Mixing corosync traffic with VM traffic causes cluster instability under load — always isolate it on its own NIC or VLAN.

---

## Config File

All network config lives in:

```
/etc/network/interfaces
```

After editing, reload with:

```bash
# Safer — only reloads changed interfaces, doesn't bounce everything
ifreload -a

# Full restart — use only if ifreload isn't available
systemctl restart networking
```

> Always make network changes via the Proxmox GUI or with console access available — a bad config will drop your SSH session.

---

## Basic Bridge (Single NIC)

The simplest setup — one physical NIC, one bridge, Proxmox management and VMs share it.

```bash
# /etc/network/interfaces

auto lo
iface lo inet loopback

auto enp1s0
iface enp1s0 inet manual

auto vmbr0
iface vmbr0 inet static
    address 192.168.1.100/24
    gateway 192.168.1.1
    bridge-ports enp1s0
    bridge-stp off
    bridge-fd 0
```

---

## VLAN-Aware Bridge

Enables VLAN tagging per VM directly in Proxmox. The bridge acts as a trunk — VMs are assigned a VLAN tag in their network config in the Proxmox GUI.

```bash
auto vmbr0
iface vmbr0 inet static
    address 192.168.1.100/24
    gateway 192.168.1.1
    bridge-ports enp1s0
    bridge-stp off
    bridge-fd 0
    bridge-vlan-aware yes
    bridge-vids 2-4094        # VLANs allowed on this bridge
```

> Switch port must be configured as a trunk allowing the same VLANs.

---

## LACP Bond + VLAN-Aware Bridge

Production pattern — bond two NICs for redundancy and throughput, VLAN-aware bridge on top for VM trunk.

**Switch side:** configure ports as LACP LAG (802.3ad), set as trunk allowing required VLANs.

```bash
# /etc/network/interfaces

auto lo
iface lo inet loopback

# Physical NICs — no IP, just members of the bond
auto enp1s0
iface enp1s0 inet manual

auto enp2s0
iface enp2s0 inet manual

# LACP bond
auto bond0
iface bond0 inet manual
    bond-slaves enp1s0 enp2s0
    bond-mode 802.3ad
    bond-miimon 100
    bond-lacp-rate fast
    bond-xmit-hash-policy layer3+4

# VLAN-aware bridge over the bond
auto vmbr0
iface vmbr0 inet static
    address 192.168.1.100/24
    gateway 192.168.1.1
    bridge-ports bond0
    bridge-stp off
    bridge-fd 0
    bridge-vlan-aware yes
    bridge-vids 10 20 30 40 50
```

---

## Storage Network (Dedicated NIC + Jumbo Frames)

Storage traffic (Ceph, iSCSI, NFS) should be on a dedicated NIC with no gateway — it's a private backend network. Jumbo frames (MTU 9000) reduce CPU overhead on high-throughput storage traffic.
> Ceph has a network config file `/etc/ceph/ceph.conf` that defines the client and replication networks. Ceph then will bind to interfaces that have IPs in the subnets.

```bash
auto enp3s0
iface enp3s0 inet manual

auto vmbr1
iface vmbr1 inet static
    address 10.10.10.1/24
    bridge-ports enp3s0
    bridge-stp off
    bridge-fd 0
    mtu 9000                  # jumbo frames — match on switch and all storage hosts
```

> No gateway on storage bridges — this is intentional. Storage traffic should never leave this network.
> The storage target should only be available on this interface. The OS will use its routing table to determine what route to use, no gateway and is only available on the storage interface prevent unexpected interface use. 

---

## Cluster Network (Corosync)

Proxmox clusters use corosync for node communication and quorum. Needs a dedicated low-latency path — mixing with VM traffic risks cluster timeouts and split-brain.

> edit `/etc/corosync/corosync.conf` file to define the hosts IP address. This "pins" corosync to that interface. 

```bash
auto enp4s0
iface enp4s0 inet static
    address 10.10.20.1/24
    bridge-stp off
    bridge-fd 0
```

```bash
# Check corosync network config
cat /etc/corosync/corosync.conf

# Check cluster status
pvecm status

# Check node communication
corosync-cfgtool -s
```

Example `corosync.config`  
```
nodelist { 
	node { 
		name: pve1 
		nodeid: 1 
		ring0_addr: 10.10.20.1 # ← this is the IP on your corosync interface 
	} 
	node { 
		name: pve2 
		nodeid: 2 
		ring0_addr: 10.10.20.2 
	} 
}
```

---

## Linux Bridge vs Open vSwitch (OVS)

Proxmox supports both. Most setups use Linux bridges.

|                        | Linux Bridge   | OVS                       |
| ---------------------- | -------------- | ------------------------- |
| Complexity             | Simple         | More complex              |
| VLAN support           | Good           | Advanced                  |
| Tunnel support (VXLAN) | Limited        | Full                      |
| SDN integration        | Yes            | Yes                       |
| Homelab use            | Default choice | Rarely needed             |
| Enterprise use         | Common         | Used for complex overlays |

> OVS is worth knowing exists if you need VXLAN tunnels or complex overlay networking — otherwise Linux bridges cover most use cases.

---

## SDN (Software Defined Networking)

Proxmox 8.x added GUI-based SDN management. Lets you manage VLANs, VXLANs, and network zones without editing `/etc/network/interfaces` directly. The direction Proxmox is heading for network management.

```bash
# SDN config lives here
/etc/pve/sdn/

# Apply SDN config changes
pvesh set /cluster/sdn
```

---

## VM Network Performance Tips

**Use Virtio drivers** — always select `virtio` as the NIC model for Linux VMs. Paravirtualized, significantly better performance than emulated e1000.

**e1000/e1000e** — emulated Intel NIC, use only for Windows VMs or OS that doesn't support virtio. Known issues with e1000e driver in some Proxmox versions causing network instability — if you see drops on a VM using e1000e, switch to virtio.

**Multi-queue** — for high-throughput VMs enable multiple network queues. Set in VM hardware config — match queue count to vCPU count.

**SR-IOV** — direct hardware access bypassing the hypervisor network stack entirely. Requires SR-IOV capable NIC and switch. Maximum performance but loses live migration ability.

---

## Verification Commands

```bash
# Show all interfaces and IPs
ip addr show

# Show bridge info and attached ports
brctl show

# Show routing table
ip route show

# Detailed bridge VLAN info
bridge vlan show

# Check bond status and active slave
cat /proc/net/bonding/bond0

# Check current network config
cat /etc/network/interfaces

# Test connectivity
ping -c 4 8.8.8.8

# Check corosync cluster network
corosync-cfgtool -s
pvecm status
```

---

## Quick Reference

```bash
ifreload -a                          # reload network config safely
brctl show                           # show bridges and members
bridge vlan show                     # show VLAN assignments on bridges
ip addr show                         # all interfaces and IPs
ip route show                        # routing table
cat /proc/net/bonding/bond0          # bond status
cat /etc/network/interfaces          # full network config
pvecm status                         # cluster node status
corosync-cfgtool -s                  # corosync link status
```