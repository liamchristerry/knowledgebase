
Tags: #cisco #fibrechannel #unity #dell #sysadmin #cheatsheet #san

## Configuration Logic

> This config is for side B of the Fibre Channel fabric. Note that only SPB is configured below — side A is handled by a separate switch, and each switch acts independently.

Create aliases for each WWPN (end device) connected to the fabric.

```
fcalias name Host1-vmhba5 vsan 200
  member pwwn 21:00:f4:c7:aa:9d:de:ae
```

Create zones. Each zone is scoped to a single host and its target storage ports. This example allows Host 1 to communicate with SPB FC ports 0–3.

```
zone name UNITY_HOST1 vsan 200
  member fcalias UNITY_SPB_FC0
  member fcalias UNITY_SPB_FC1
  member fcalias UNITY_SPB_FC2
  member fcalias UNITY_SPB_FC3
  member fcalias Host1-vmhba5
  member fcalias Host1-vmhba6
```

Create a zone set and add zones to it. The zone set ties all zones together and defines what is active in the fabric.

```
zoneset name BTC_MDF_MDS_Bottom vsan 200
  member UNITY_HOST1
  member UNITY_HOST2
  member UNITY_HOST3
  member UNITY_HOST4
```

Activate the zone set. This pushes the configuration live into the fabric.

```
zoneset activate name BTC_MDF_MDS_Bottom vsan 200
```

---

## How to Configure

- Use the FLOGI database to confirm the WWPNs of all connected devices.
- Define which ports will be used and which VSAN they belong to.
    - A VSAN is conceptually similar to a VLAN — logical fabric segmentation.
- Ports are in a shutdown/disabled state by default.
- Create FCAlias entries to map each WWPN to a human-readable name.
- Create zones and add FCAlias entries to them.
    - Each zone should contain a **single host (initiator)** and its target ports. Do not create large shared zones.
- Create a zone set and add zones to it.
    - Only one zone set per VSAN can be active at a time.
    - To make changes, update the zone set and re-activate it.

---

## Full Configuration Example

```
config t

! Create VSAN
vsan database
  vsan 200

! Assign interfaces to VSAN
! Note: Bulk assignment is convenient but explicit per-port config reduces misconfiguration risk.
vsan database
  vsan 200 interface fc1/1-24

! --- FCAlias Definitions ---

! Host 1
fcalias name Host1-vmhba5 vsan 200
  member pwwn 21:00:f4:c7:aa:9d:de:ae

fcalias name Host1-vmhba6 vsan 200
  member pwwn 21:00:f4:c7:aa:9d:de:af

! Host 2
fcalias name Host2-vmhba5 vsan 200
  member pwwn 21:00:f4:c7:aa:9e:b1:de

fcalias name Host2-vmhba6 vsan 200
  member pwwn 21:00:f4:c7:aa:9e:b1:df

! Host 3
fcalias name Host3-vmhba5 vsan 200
  member pwwn 21:00:f4:c7:aa:9e:7d:5e

fcalias name Host3-vmhba6 vsan 200
  member pwwn 21:00:f4:c7:aa:9e:7d:5f

! Host 4
fcalias name Host4-vmhba5 vsan 200
  member pwwn 21:00:f4:c7:aa:9a:42:be

fcalias name Host4-vmhba6 vsan 200
  member pwwn 21:00:f4:c7:aa:9a:42:bf

! Dell Unity — Storage Processor B (SPB)
fcalias name UNITY_SPB_FC0 vsan 200
  member pwwn 50:06:01:6C:47:E0:7F:A6

fcalias name UNITY_SPB_FC1 vsan 200
  member pwwn 50:06:01:6D:47:E0:7F:A6

fcalias name UNITY_SPB_FC2 vsan 200
  member pwwn 50:06:01:6E:47:E0:7F:A6

fcalias name UNITY_SPB_FC3 vsan 200
  member pwwn 50:06:01:6F:47:E0:7F:A6

! --- Zone Definitions ---

zone name UNITY_HOST1 vsan 200
  member fcalias UNITY_SPB_FC0
  member fcalias UNITY_SPB_FC1
  member fcalias UNITY_SPB_FC2
  member fcalias UNITY_SPB_FC3
  member fcalias Host1-vmhba5
  member fcalias Host1-vmhba6

zone name UNITY_HOST2 vsan 200
  member fcalias UNITY_SPB_FC0
  member fcalias UNITY_SPB_FC1
  member fcalias UNITY_SPB_FC2
  member fcalias UNITY_SPB_FC3
  member fcalias Host2-vmhba5
  member fcalias Host2-vmhba6

zone name UNITY_HOST3 vsan 200
  member fcalias UNITY_SPB_FC0
  member fcalias UNITY_SPB_FC1
  member fcalias UNITY_SPB_FC2
  member fcalias UNITY_SPB_FC3
  member fcalias Host3-vmhba5
  member fcalias Host3-vmhba6

zone name UNITY_HOST4 vsan 200
  member fcalias UNITY_SPB_FC0
  member fcalias UNITY_SPB_FC1
  member fcalias UNITY_SPB_FC2
  member fcalias UNITY_SPB_FC3
  member fcalias Host4-vmhba5
  member fcalias Host4-vmhba6

! --- Zone Set ---

zoneset name BTC_MDF_MDS_Bottom vsan 200
  member UNITY_HOST1
  member UNITY_HOST2
  member UNITY_HOST3
  member UNITY_HOST4

! Activate the zone set
zoneset activate name BTC_MDF_MDS_Bottom vsan 200

! Save configuration
copy running-config startup-config
```

---

## Troubleshooting

### Check connected WWPNs and their switch ports

Use the FLOGI database to see what devices have logged into the fabric and which physical port they are on. This is useful for catching A/B side port mix-ups.

```
show flogi database
```

> [!note] WWPNs are not tied to specific switch ports. Zoning is WWPN-based, while ports are only assigned to a VSAN. This means devices can be moved between ports without changing zoning.
> 
> If a device is not present in the FLOGI database, you have a physical connectivity issue. FLOGI login is the first step in the FC connection process.

### Check interface state and VSAN assignment

```
show interface brief
show interface fc1/1
```

### Confirm the target or initiator is visible in the fabric name server

```
show fcns database
```

### Verify active zones and zone sets

```
show zone active vsan 200
show zoneset active vsan 200
```

### Verify a specific FCAlias

```
show fcalias name <alias> vsan 200
```

### Check error counters on an interface

```
show interface fc1/1 counters
show interface fc1/1 | inc error
```