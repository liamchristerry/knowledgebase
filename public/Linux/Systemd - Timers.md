
## Overview

systemd timers are the modern replacement for cron. Always a pair — a `.timer` unit that defines the schedule, and a matching `.service` unit that does the actual work. Same base name required.

```
backup.timer   →   triggers   →   backup.service
```

### Timers vs Cron

|cron|systemd timer|
|---|---|---|
|Logging|No built-in|Full journald logging|
|Missed runs|Lost if system was off|Caught up with `Persistent=true`|
|Dependencies|None|Full After=/Requires= support|
|On-boot delay|No|Yes, `OnBootSec=`|
|Resource limits|No|Full cgroup controls on service|

---

## Timer Unit Template

```ini
# /etc/systemd/system/backup.timer

[Unit]
Description=Run backup daily at 2am

[Timer]
# Run at 2am every day
OnCalendar=*-*-* 02:00:00

# If the system was off at scheduled time, run on next boot
Persistent=true

# Optional — delay after boot before first run
# Useful to let system settle before doing heavy work
OnBootSec=5min

# Optional — spreads load if many timers fire at once
RandomizedDelaySec=30

[Install]
WantedBy=timers.target
```

---

## Matching Service Unit

```ini
# /etc/systemd/system/backup.service

[Unit]
Description=Backup script

[Service]
Type=oneshot
ExecStart=/usr/local/bin/backup.sh
User=liam
StandardOutput=journal
StandardError=journal
```

> The service has no `[Install]` section — it is only ever triggered by the timer, never enabled directly

---

## OnCalendar Schedule Syntax

### Format

```
DayOfWeek  Year-Month-Day  Hour:Minute:Second
```

Use `*` as a wildcard for any value.

### Common Examples

```bash
# Every day at 2am
OnCalendar=*-*-* 02:00:00

# Every hour
OnCalendar=hourly

# Every day at midnight
OnCalendar=daily

# Every week (Monday midnight)
OnCalendar=weekly

# Every month (1st at midnight)
OnCalendar=monthly

# Every 15 minutes
OnCalendar=*:0/15

# Monday to Friday at 9am
OnCalendar=Mon..Fri *-*-* 09:00:00

# Every Sunday at 3:30am
OnCalendar=Sun *-*-* 03:30:00

# First day of every month at 6am
OnCalendar=*-*-01 06:00:00
```

### Shorthand Keywords

```bash
OnCalendar=minutely       # every minute
OnCalendar=hourly         # every hour
OnCalendar=daily          # every day at midnight
OnCalendar=weekly         # every Monday midnight
OnCalendar=monthly        # 1st of month midnight
OnCalendar=yearly         # Jan 1st midnight
OnCalendar=quarterly      # Jan/Apr/Jul/Oct 1st
```

---

## Test Your Schedule Before Deploying

Always verify a calendar expression before enabling a timer on anything important:

```bash
# Check when an expression will next fire
systemd-analyze calendar "Mon..Fri *-*-* 09:00:00"
```

Example output:

```
  Original form: Mon..Fri *-*-* 09:00:00
Normalized form: Mon..Fri *-*-* 09:00:00
    Next elapse: Mon 2025-01-20 09:00:00 EST
       (in UTC): Mon 2025-01-20 14:00:00 UTC
       From now: 14h left
```

Shows the normalized form, next trigger time, and how long until it fires. Run this every time.

---

## Monotonic Timers (Relative to Events, Not Clock)

Alternative to `OnCalendar` — fires relative to system events rather than wall clock time.

```ini
[Timer]
# 15 minutes after boot
OnBootSec=15min

# 30 minutes after the service last ran
OnUnitActiveSec=30min

# 10 minutes after the service last finished
OnUnitInactiveSec=10min
```

Useful for "run a health check 10 minutes after the last one finished" rather than at a fixed clock time.

---

## Managing Timers

```bash
# Reload after creating/editing unit files
sudo systemctl daemon-reload

# Enable and start the timer (not the service)
sudo systemctl enable --now backup.timer

# Check timer status and next run time
systemctl status backup.timer

# List all active timers with next/last run times
systemctl list-timers --all

# Run the service manually right now, bypassing the timer
sudo systemctl start backup.service

# Check service logs
journalctl -u backup.service
journalctl -u backup.service -f      # follow live
journalctl -u backup.service -b      # since last boot only
```

---

## Quick Reference

```bash
systemd-analyze calendar "*-*-* 02:00:00"    # test schedule expression
systemctl enable --now backup.timer          # enable and start timer
systemctl list-timers --all                  # all timers and next run time
systemctl start backup.service               # run service manually now
journalctl -u backup.service                 # view service logs
systemctl status backup.timer                # timer status and next trigger
```