Tags: #linux #ubuntu #sysadmin #updates #automation

---
### Log Files

```bash
/var/log/unattended-upgrades/unattended-upgrades/unattended-upgrades.log
# gives a highlight of when updates were checked 
/var/log/unattended-upgrades/unattended-upgrades/unattended-upgrades-dpkg.log
# what packages were installed and what services were rebooted
```

### What will be updated
- By Default
	- Kernel 
	- Security patches for the OS
- What can be updated
	- Applications in main ubuntu repos
	- I think you can add 3rd party repos but not really suggested since these will not publish security updates that separate. Security updates are bundled in with software versions



## Example Files

> /etc/apt/apt.conf.d/50unattended-upgrades
```ini
Unattended-Upgrade::Allowed-Origins {
// Base Ubuntu release packages - as shipped on install day, rarely changes
"${distro_id}:${distro_codename}";
  

// Security patches only - CVEs and critical fixes, low risk, fast turnaround
"${distro_id}:${distro_codename}-security";


// Stable bug fixes and broader updates - tested before release, still low risk
// Remove this line if you want security-only updates
"${distro_id}:${distro_codename}-updates";
};


// NOTE: Only packages from Ubuntu's official repos are updated with this config.
// Third-party repos (Docker, Hashicorp, PPAs etc) are never touched - update manually
// or via a separate weekly apt upgrade cron job.
// noble-backports and noble-proposed are intentionally excluded - too risky for auto-update


// Sends logs to syslog /var/log/syslog
Unattended-Upgrade::SyslogEnable "true";
Unattended-Upgrade::SyslogFacility "daemon";


// Auto reboot 3am if users are logged in or not
Unattended-Upgrade::Automatic-Reboot "true";
Unattended-Upgrade::Automatic-Reboot-Time "03:00";


// Cleanup
Unattended-Upgrade::Remove-Unused-Dependencies "true";
Unattended-Upgrade::Remove-Unused-Kernel-Packages "true";
Unattended-Upgrade::LogRotate "6";

Unattended-Upgrade::Package-Blacklist {
};

// Normal Unattended Update logs /var/log/unattended-upgrades/
```

> /etc/apt/apt.conf.d/20auto-upgrades
```ini
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Unattended-Upgrade "1";
APT::Periodic::AutocleanInterval "7";
```