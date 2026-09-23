Tags: #linux #ubuntu #logging #sysadmin #journald #syslog #sysadmin 

---

## Overview

Linux logging comes from two parallel systems that coexist on modern Ubuntu:

- **journald** — systemd's binary logging system, captures everything from boot, all services, kernel. Primary log system on modern Ubuntu
- **syslog (rsyslog)** — traditional text-based logging, writes to `/var/log/`. Still present and useful, especially for compatibility and forwarding logs to remote systems
	- lots of 2000's linux services and applications use syslog

---

## journald

### Key Commands

```bash
# Follow all logs live (like tail -f for everything)
journalctl -f

# Follow a specific service
journalctl -u sshd -f

# All logs for a specific service
journalctl -u nginx

# Logs since last boot only
journalctl -b

# Logs from previous boot (useful after a crash)
journalctl -b -1

# List available boots
journalctl --list-boots

# Kernel messages only (equivalent to dmesg)
journalctl -k

# Filter by priority
# emerg/alert/crit/err/warning/notice/info/debug
journalctl -p err

# Logs between time range
journalctl --since "2024-01-01 00:00:00" --until "2024-01-02 00:00:00"

# Logs since X time ago
journalctl --since "1 hour ago"
journalctl --since "yesterday"

# Show most recent N lines
journalctl -n 50

# Show logs for current boot with errors and above
journalctl -b -p err

# Output as JSON (useful for parsing/scripting)
journalctl -u nginx -o json-pretty

# Disk usage of journal
journalctl --disk-usage


# Filter by systemd unit
journalctl _SYSTEMD_UNIT=nginx.service

# Filter by PID
journalctl _PID=1234

# Filter by executable
journalctl _EXE=/usr/sbin/sshd

# Filter by user ID
journalctl _UID=1000

# Combine filters
journalctl _SYSTEMD_UNIT=sshd.service _PID=1234

# See all available fields for a log entry
journalctl -o verbose -n 1

# Export to JSON for parsing/scripting
journalctl -u nginx --since today -o json | jq '.MESSAGE'
```

### journald Storage

journald logs live in `/var/log/journal/` if persistent logging is enabled, or `/run/log/journal/` if not (lost on reboot).

```bash
# Check if persistent logging is enabled
cat /etc/systemd/journald.conf | grep Storage

# Enable persistent logging
sudo mkdir -p /var/log/journal
sudo systemctl restart systemd-journald
```

### journald Config — `/etc/systemd/journald.conf`

```ini
[Journal]
# persistent = survive reboot, volatile = RAM only, auto = persistent if dir exists
Storage=persistent

# Max disk space journal can use
SystemMaxUse=500M

# Max size of individual journal file
SystemMaxFileSize=50M

# How long to keep old journal files
MaxRetentionSec=1month

# How to use Journald native forwarding. Sends to Syslog socket or something, duplicate logs can be generated using this so look into
ForwardToSyslog=yes
```

After changes:

```bash
sudo systemctl restart systemd-journald
```

---

## syslog / rsyslog

Traditional text log files written to `/var/log/`. rsyslog is the daemon that manages these on Ubuntu. Still relevant because:

- Human readable, no special command needed to read
- Easy to forward to remote log servers (SIEM, Graylog, Splunk)
- Some older applications only write to syslog

### Key Log Files

| File                       | Contents                                       |
| -------------------------- | ---------------------------------------------- |
| `/var/log/syslog`          | General system messages, catch-all             |
| `/var/log/auth.log`        | Authentication — sudo, su, SSH logins/failures |
| `/var/log/kern.log`        | Kernel messages                                |
| `/var/log/dpkg.log`        | Package install/remove history                 |
| `/var/log/apt/history.log` | apt command history                            |
| `/var/log/fail2ban.log`    | fail2ban actions                               |

### Reading Log Files

```bash
# Follow auth log live
sudo tail -f /var/log/auth.log

# Search for a pattern
sudo grep "Failed password" /var/log/auth.log

# Search across compressed rotated logs too
sudo zgrep "Failed password" /var/log/auth.log*

# Last N lines
sudo tail -n 100 /var/log/auth.log
```

### rsyslog Config

Main config at `/etc/rsyslog.conf` with drop-ins at `/etc/rsyslog.d/`.

rsyslog uses facility.severity format to route messages to files:
Facility examples

|Facility|What it covers|
|---|---|
|`auth`|Authentication — sudo, su|
|`authpriv`|Private auth messages — SSH, PAM|
|`kern`|Kernel messages|
|`daemon`|Background system daemons|
|`cron`|Cron job messages|
|`mail`|Mail server messages|
|`user`|Generic user-level messages|
|`local0-local7`|Reserved for custom app use|

```ini
# Any message from auth OR authpriv facility, any severity
# → write to auth.log
auth,authpriv.*    /var/log/auth.log

# Any message from the kernel, any severity
# → write to kern.log
kern.*             /var/log/kern.log

# Any message from ANY facility, info severity or above
# → write to syslog (this is the catch-all)
*.info             /var/log/syslog
```

---

## Login History Files - Binary Files

> Binary files cant be read by `cat`, the below commands are how you read the logs
### wtmp — Successful Logins

```bash
# Show last successful logins
last

# Show last logins with full dates and hostnames
last -aF

# Show last logins for a specific user
last liam

# Show last system reboots
last reboot

# Show last shutdowns
last -x shutdown
```

### btmp — Failed Login Attempts

```bash
# Show failed login attempts
sudo lastb

# Full output with hostname and full date
sudo lastb -aF

# Limit output
sudo lastb -n 20
```

### lastlog — Last Login Per User

```bash
# Show last login time for every user on the system
lastlog

# Specific user
lastlog -u liam
```

---

## Kernel & Hardware Logs

### dmesg

Kernel ring buffer — hardware detection, driver messages, kernel errors. Most useful right after boot or when troubleshooting hardware issues.

> Kernel Ring Buffer - fixed size RAM chuck that kernel writes to during boot and I believe the entire time the machine is running. Older logs are overwritten when its full (why its called a ring). 

```bash
# View kernel messages
dmesg

# Follow live (useful when plugging in hardware)
dmesg -w

# Human readable timestamps
dmesg -T

# Filter by log level
dmesg --level=err,warn

# Filter for specific hardware (grep)
dmesg | grep -i usb
dmesg | grep -i eth
dmesg | grep -i nvme
dmesg | grep -i error

# Via journald (same content, better filtering)
journalctl -k
journalctl -k -b -1    # kernel messages from previous boot
```

---

## Security & Auth Log Patterns

Useful grep patterns for auth.log — things worth checking regularly on any internet-facing server:

```bash
# Failed SSH password attempts
sudo grep "Failed password" /var/log/auth.log

# Successful SSH logins
sudo grep "Accepted" /var/log/auth.log

# All sudo usage
sudo grep "sudo" /var/log/auth.log

# Invalid users attempting login
sudo grep "Invalid user" /var/log/auth.log

# Account lockouts
sudo grep "pam_faillock" /var/log/auth.log
# or for older systems replace `pam_faillock` for `pam_tally`
# SSH disconnections
sudo grep "Disconnected" /var/log/auth.log
```

---

## Log Rotation — logrotate

Log files would grow forever without rotation. `logrotate` handles compressing and cycling old logs on a schedule.

```bash
# Config files
/etc/logrotate.conf          # main config
/etc/logrotate.d/            # per-app drop-ins

# Test rotation config without actually rotating
sudo logrotate --debug /etc/logrotate.conf

# Force rotation now
sudo logrotate --force /etc/logrotate.conf
```

Example logrotate config for a custom app:

```
/var/log/myapp/*.log {
    daily               # rotate daily
    rotate 14           # keep 14 days
    compress            # gzip old logs
    delaycompress       # compress on second rotation
    missingok           # don't error if log missing
    notifempty          # skip if log is empty
    create 0640 www-data adm   # permissions on new file
}
```

---

## Remote Logging (rsyslog Forwarding)

Useful for centralising logs from multiple servers to a SIEM or log aggregator like Graylog, Splunk, or even just another syslog server.

```bash
# In /etc/rsyslog.d/99-remote.conf

# Forward everything via UDP
*.* @192.168.1.100:514

# Forward everything via TCP (more reliable)
*.* @@192.168.1.100:514

# Forward only auth logs
auth,authpriv.* @@192.168.1.100:514
```

```bash
sudo systemctl restart rsyslog
```

> NOTE - above configuration for sending over UDP or TCP is unencrypted with no security. `rsyslog-gnutls` application installs the needed drivers to do TLS. But added configuration is needed like certificates
---

## Useful One-Liners

```bash
# Top 10 IPs with failed SSH attempts
sudo grep "Failed password" /var/log/auth.log | awk '{print $11}' | sort | uniq -c | sort -rn | head

# Count failed logins by username
sudo grep "Failed password" /var/log/auth.log | awk '{print $9}' | sort | uniq -c | sort -rn

# Watch auth log and highlight failures
sudo tail -f /var/log/auth.log | grep --line-buffered "Failed\|Invalid\|error"

# Journal errors from this boot across all services
journalctl -b -p err --no-pager

# Disk space used by all logs
du -sh /var/log/*
```

---

## Quick Reference

```bash
journalctl -f                          # follow all logs live
journalctl -u nginx -f                 # follow specific service
journalctl -b -p err                   # errors this boot
journalctl -b -1                       # previous boot logs
journalctl -k                          # kernel messages
journalctl --disk-usage                # journal disk usage
sudo tail -f /var/log/auth.log         # follow auth log
sudo tail -f /var/log/syslog           # follow syslog
last -aF                               # successful logins full detail
sudo lastb -aF                         # failed logins full detail
lastlog                                # last login per user
dmesg -T                               # kernel messages with timestamps
dmesg -w                               # follow kernel messages live
sudo grep "Failed password" /var/log/auth.log   # SSH failures
```


---
## Auditd

```bash
sudo apt install auditd audispd-plugins

# Check status
sudo systemctl status auditd

# Live audit log
sudo tail -f /var/log/audit/audit.log
```

Audit rules examples
```bash
# Watch a specific file for any access
sudo auditctl -w /etc/passwd -p rwxa -k passwd_changes

# Watch a directory
sudo auditctl -w /etc/sudoers.d/ -p wa -k sudoers_changes

# Log all commands run by a specific user
sudo auditctl -a always,exit -F arch=b64 -F uid=1000 -S execve -k user_commands

# View current rules
sudo auditctl -l

# Search audit log for a key
sudo ausearch -k passwd_changes

# Generate a human readable report
sudo aureport --summary
sudo aureport --auth        # authentication report
sudo aureport --login       # login report
sudo aureport --failed      # failed events
```

>Persistent rules go in `/etc/audit/rules.d/` — anything set with `auditctl` alone is lost on reboot.


---

Boot performance

```bash
# How long did boot take, broken down by service
systemd-analyze

# Full blame list — which services took longest
systemd-analyze blame

# Visual waterfall chart of boot sequence
systemd-analyze plot > boot.svg

# Critical chain — what was on the critical path
systemd-analyze critical-chain
```


---

## Log Integrity and Tamper Detection

A gap that matters on anything security-sensitive — logs are only useful for forensics if you can trust they haven't been tampered with.

**journald FSS (Forward Secure Sealing)** — cryptographically seals journal entries so tampering is detectable:

bash

```bash
# Set up sealing keys
sudo journalctl --setup-keys

# Verify journal integrity
sudo journalctl --verify
```

**Offsite/remote logging** — the most practical tamper protection. If logs are forwarded to a remote server in real time, an attacker compromising the local machine can't retroactively clean the remote copy. This is the main reason centralised logging matters in a security context, not just convenience.