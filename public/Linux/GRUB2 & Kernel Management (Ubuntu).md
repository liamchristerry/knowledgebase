Tags: #linux #grub #kernal #boot #ubuntu #updates #sysadmin 

---

## Older Ubuntu Versions
	- https://old-releases.ubuntu.com/releases/
## Boot Order

```
BIOS/UEFI
    ▼
GRUB2  ← reads /boot/grub/grub.cfg - selects the kernal to load
    ▼
Kernel (vmlinuz)
    ▼
initramfs  ← temporary root filesystem, loads drivers
    ▼
systemd  ← PID 1, brings up the rest of the system
    ▼
Your system (login prompt / desktop)
```

## Key Files

| File                  | Purpose                                    |
| --------------------- | ------------------------------------------ |
| `/boot/grub/grub.cfg` | Live GRUB config — **never edit directly** |
| `/etc/default/grub`   | Your config — edit this                    |
| `/boot/vmlinuz-*`     | Kernel images                              |
| `/boot/initramfs-*`   | One initramfs per kernel                   |

> Always edit `/etc/default/grub`, then run `update-grub` to regenerate `grub.cfg`.


## Check Current Kernel

```bash
# What kernel is running right now
uname -r

# All installed kernels
dpkg --list | grep linux-image
```

## Kernel Tracks
- HWE (hardware Enabled) - installed on desktop OS's and any Ubuntu that is not LTS like 23.04 for example. 
	- This is used to take advantage of newer drivers that the GA kernels might not have 
- GA (General Availability) - Installed on LTS server OS's. This kernel version number does not change. Example 6.8 is the Kernel for 24.04 Server LTS. When you update the OS and the kernel ABI (bold below) changes but not the kernel version 
	- 6.8.12-**101**-generic
## Check for Available Kernel Updates

```bash
# See if kernel updates are available (doesn't install)
apt list --upgradable | grep linux-image

# Or do a dry run
apt upgrade --dry-run | grep linux-image
```


## Apply a Kernel Upgrade

```bash
sudo apt update
sudo apt upgrade

# Reboot to load the new kernel
sudo reboot

# After reboot — confirm new kernel is active
uname -r
```

> When a new kernel installs, `update-grub` runs automatically and adds it to the boot menu. Old kernels are kept as fallbacks (usually last 2).


## Pin / Hold a Kernel (Prevent Upgrade)

Useful on servers where you want to stay on a known-good kernel.

```bash
# Hold the current kernel (replace with your version)
sudo apt-mark hold linux-image-$(uname -r) linux-headers-$(uname -r)

# Verify what's held
apt-mark showhold

# Release the hold when ready to upgrade
sudo apt-mark unhold linux-image-$(uname -r) linux-headers-$(uname -r)
```


## Fall Back to an Older Kernel

### One-time (at boot)

At the GRUB menu → **Advanced options for Ubuntu** → select older kernel. This doesn't change the default, just boots it once.

> **Tip:** If you're missing the GRUB menu, hold `Shift` during boot (BIOS) or tap `Esc` (UEFI).
> **Or** you can edit the `/etc/default/grub` and add a grub splash timeout

### Permanently set an older kernel as default

```bash
# See available kernels and their GRUB menu names
# sudo does not work? has to be ran as root
grep menuentry /boot/grub/grub.cfg

# Edit /etc/default/grub and set GRUB_DEFAULT to the submenu path
# Example: To determine what this line needs to be see below, already clicked "Advanced options for Ubuntu"
# Take a backup of the file before editing
GRUB_DEFAULT="Advanced options for Ubuntu>Ubuntu, with Linux 6.8.0-49-generic"

# Rebuild grub config
sudo update-grub

# Reboot and verify
sudo reboot
uname -r
```
![[Pasted image 20260316132123.png]]
### Remove a bad kernel entirely

```bash
sudo apt remove linux-image-5.15.0-91-generic
sudo update-grub

# Might be worth running to clean up any of the newer modules that came with the kernal
sudo apt autoremove
```

### Reinstall after removing kernal
After APT installs and you remove the package the best way to reinstall it is to just manually do it in apt. Apt update does not pick it up 
```
sudo apt install linux-image-generic
```

## GRUB Config Tweaks (`/etc/default/grub`)

Always run `sudo update-grub` after any changes here.

```bash
# Always show the boot menu (don't hide it)
##### Very useful
GRUB_TIMEOUT_STYLE=menu
GRUB_TIMEOUT=10

# Default kernel to boot (0 = first/newest, or use full menu entry name)
GRUB_DEFAULT=0

# Kernel parameters passed at boot
GRUB_CMDLINE_LINUX="quiet splash"
```

After editing:

```bash
sudo update-grub
```


## Useful Kernel Parameters

Add these to `GRUB_CMDLINE_LINUX` in `/etc/default/grub`, then run `sudo update-grub`.

|Parameter|When to use|
|---|---|
|`quiet splash`|Default desktop — suppress boot messages, show splash|
|`nomodeset`|Black screen on boot, GPU driver issues|
|`intel_iommu=on`|Enabling PCIe passthrough for VMs (Intel CPU)|
|`amd_iommu=on`|Same, for AMD CPU|
|`transparent_hugepage=never`|Running Redis, MongoDB, or Oracle DB|
|`panic=30`|Auto-reboot 30s after kernel panic (good for servers)|
|`systemd.unit=rescue.target`|Boot into rescue mode (one-time, set at GRUB prompt)|

### Test a parameter one-time before making it permanent

At the GRUB menu, highlight your kernel entry → press `e` → find the `linux` line → append your param at the end → `Ctrl+X` to boot. Changes don't persist.

### Check what params the running kernel was booted with

```bash
cat /proc/cmdline
```



