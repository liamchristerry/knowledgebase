Tags: #linux #netplan #bonding #networking #ubuntu #sysadmin 

---
## Creating a bond
Example below of a Netplan config on a Ubuntu physical server
- **bond0**: is used as the bonded interface
- **enp** - Interfaces are the Fiber cards
- **eno** - Interfaces are the integrated Ethernet ports on the server motherboard

Using 802.3ad (LCAP) for the load balancing. The switch needs to be configured accordingly
```
# This is the network config written by 'subiquity'
network:
  bonds:
    bond0:
      addresses: [x.x.x.x/24]
      gateway4: x.x.x.x
      interfaces:
      - enp4s0f0
      - enp6s0f0
      parameters:
        mode: 802.3ad
      dhcp4: false
      optional: true
      nameservers:
        addresses: [x.x.x.x, x.x.x.x]
  ethernets:
    eno1:
      dhcp4: true
    eno2:
      dhcp4: true
    eno3:
      dhcp4: true
    eno4:
      dhcp4: true
    enp4s0f0:
      dhcp4: false
      optional: true
    enp4s0f1:
      dhcp4: true
    enp6s0f0:
      dhcp4: false
      optional: true
    enp6s0f1:
      dhcp4: true
  version: 2
```

### Commands 
```
ethtool bond0
# Will show you information about the bond including the speed. 

ip address
# Shows the IP, MAC, status of the interfaces
# Use this command to show interfaces names

networkctl list
# Shows that the interfaces assigned to the bond are bonded

cat /proc/net/bonding/bond0
# Shows some more general information about the the bond and its assigned interfaces

netplan try
# Big one, this will attempt to apply the config but if you do not confirm its working after 60 seconds I think it will revert
```



