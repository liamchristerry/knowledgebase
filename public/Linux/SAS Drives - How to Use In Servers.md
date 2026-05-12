Tags: #linux #sas #harddrives #block #sysadmin

---

## References
https://forum.level1techs.com/t/how-to-reformat-520-byte-drives-to-512-bytes-usually/133021

## Drive Block Size

Disks that are used in storage arrays might be formatted in 520 Byte blocks and not 512 you normally will see on servers/PCs. 

When you plug in a drive that is formatted in 520 the OS will see the disk but cant use it. Below describes the process on converting the disks block size to something useable. 

> Worth noting that the production lab at SE does have hardware devices that will do this for you. And will report out health of the drive and attempt to fix any issues found. 
## Commands

```
lsblk
# This will show the drives but since they are not readable right now they dont show much information besides there drive location

sudo apt install sg3_utils
# Install the SG tools to modify the drives block size

sudo sg_format -l /dev/sdx
# This will tell you the logical block size
```

> In the link they talk about an "illegal command", I never ran into that luckily. I just formatted them

## Convert the Block Size

> I read someplace that its suggested to run this on only a few drives (2 or 3 max) at a time to prevent HBA overloading/overheating. Did not test this personally


> # Tip - great use of the screen command to open a new session so you can run this on multiple drives
> 
```
sudo sg_format --format --size=512 /dev/sdx
```


