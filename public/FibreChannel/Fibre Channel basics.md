# Fibre Channel Concepts

Tags: #cisco #fibrechannel #san #sysadmin #cheatsheet

---

## Core Terminology

**WWPN — World Wide Port Name** Identifies a specific port on an HBA or storage array. Used for zoning and fabric login. Analogous to a MAC address on a NIC.

**WWNN — World Wide Node Name** Identifies the whole device (the HBA card or storage controller), not a specific port. A single device will have one WWNN and multiple WWPNs.

**Fabric** All connected Fibre Channel switches acting as one logical network. Devices log into the fabric rather than directly to each other.

- Fabric = shared control plane

**FLOGI — Fabric Login** The login process by which a WWPN joins the fabric.

How it works:

1. Device sends a FLOGI request to the switch
2. Switch assigns an **FCID** (analogous to an IP address)
3. Device registers itself in the Name Server

**FCID — Fibre Channel ID** A 24-bit address assigned to a device during FLOGI. Used for routing frames across the fabric. Scoped to the fabric — similar in role to an IP address on an IP network.

**Name Server (FCNS)** A database of all devices currently logged into the fabric. Built automatically after FLOGI completes. Used by hosts and storage to discover what devices exist and where they are.

```
show fcns database
```

---

## Zoning Philosophies

### WWPN Zoning (Best Practice)

Zoning is based on device identity (WWPN) rather than physical port. Cables can be moved between switch ports without changing zone configuration, as long as the port is assigned to the correct VSAN.

See configuration example: [[Cisco MDS Configuration]]

### Port Zoning

Zoning is based on the physical Fibre Channel switch port. Simpler to reason about but fragile — moving a cable breaks zoning. Generally not recommended.

---

## Key Concepts

**Zone Sets** A zone set is a named collection of zones. Only one zone set can be active per VSAN at a time. Zone configurations exist in a pending/candidate state separately from the active state — you must explicitly activate a zone set to push it live. This is a notable difference from typical IP networking, where config changes take effect immediately.

**VSAN — Virtual SAN** Logical segmentation of a Fibre Channel fabric, conceptually similar to a VLAN. A port belongs to exactly one VSAN. A zone belongs to exactly one VSAN. VSANs on different switches that share the same ID form a single logical fabric segment.

**Initiator** The HBA on the host (server) side. Initiates storage I/O.

**Target** The storage port on the array. Receives and responds to I/O.

**Multipathing** Multiple independent physical paths between an initiator and a target. Provides redundancy and can provide load balancing depending on the implementation. Path behavior (active/active vs. active/passive) is determined by software, and differs by platform:

|Platform|Technology|
|---|---|
|VMware|NMP (Native Multipathing), PSPs|
|Dell Unity|ALUA|
|Linux|Device Mapper Multipath (dm-multipath)|

---

## Cisco vs. Brocade Terminology

|Concept|Cisco|Brocade|
|---|---|---|
|Port/device alias|`fcalias`|`alias`|
|Collection of zones|`zoneset`|`cfg`|
|Make config active|`activate`|`enable`|

---

## Connection Flow

```
Device powers on
    → FLOGI  (device joins fabric, receives FCID)
    → PLOGI  (device registers in name server)
    → Zoning filters visibility between initiators and targets
    → Host logs into storage target (session established)
    → Multipathing software builds and manages paths
```