Tags: #gpu #dell #poweredge #idrac


## Overview 
- Server Model: **Dell PowerEdge XC740xd**
- Use Case: Hyperconverged infrastructure (originally for Nutanix)
- Current OS: Proxmox
- GPUs < 75W can run using just PCI
- GPUs > 75Ws need GPU enablement kits

## Goals
- Run mixture of GPUs for PCI passthrough
- Determine what is needed for GPUs > 75W power requirements. 

## GPU Enablement Kits
[490-BEIX - Refurbished - DELL GPU Enablement KIT](https://www.itcreations.com/product/111279?srsltid=AfmBOoq5HtRlFIoj7AES4qr4bhJwJJ9ZBSOx_KH72a6vyHWh_CYuaAO6)

> I ordered a GPU enablement kit from a reseller from work purchasing team. What I was expecting to receive vs what I got is different

What I expected to be included
- High Performance fans
- Power Cables, connect riser board to GPUs
	- Straight and Y cables
- 1 U server heat sinks
- GPU enablement shroud
- Risers 1,2 and 3. I expected these for the added power ports for GPUs

What I received
- High Performance Fans
- GPU enablement shroud
- Power Cables, connect riser board to GPUs
	- Y Cables

## Get it working (OS & iDrac Detection)
Since I did not receive all the pieces I was expecting I originally tried to use just the HP fans with a standard shroud. 

> Shroud is the plastic air deflector that sits over the Memory and CPUs

This worked since riser 1 had power available already, server booted and the OS (proxmox) detected the hardware
```
# Replalce grid with whatever GPU you have
lspci | grep -i grid
```
iDrac did not detect the GPUs, it could not determine what was installed or what tempiture the cards were. After poking around online and talking to other people at work the idea of trying a standard 1U heatsink from a R640 and then use the shroud that came with the GPU enablement kit. 

Installed the heatsinks and shroud, OS detected the GPUs still and no iDrac could see/manage the GPU. 

## Shroud
The GPU enabled shroud has a few pieces
- Air blockers to force air to only open slots. I think the idea is to block air to risers that dont have GPUs. 
- Air separator that sends some air down to the CPU and some up to the GPUs

