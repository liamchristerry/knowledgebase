
## Types of Units
- service
	- runs a program
- timer
	- Cron Job alternative
	- points to a service
- socket
	- opens/listens to a TCP port
	- points to a service
- mount
	- FStab alternative
- automount
	- mounts path only if user/program access mount point
- path
	- Watches a file/directory for changes
	- points to a service

## Managing Units

```bash
systemctl start nginx
systemctl stop nginx
systemctl enable nginx
systemctl disable nginx
systemctl restart nginx
systemctl reload nginx
# Reload will have nginx reload its config, not all changes will take effect but its a way to update configs without kicking people off
```

## Systemd Directories

> Systemd uses priories for directories. A unit file in a higher priority directory will take effect over lower priority directories
```bash
/etc/systemd/system
# Best Pratice to put custom unit files here

/run/systemd/system
# Runtime units that only exisit for this currnt boot session

/lib/systemd/system
# Where package managers put unit files
```

## Structure of a Unit File
> Below is nginx unit file and only some of the options you can use, [find more here](https://www.freedesktop.org/software/systemd/man/latest/systemd.unit.html?__goaway_challenge=meta-refresh&__goaway_id=2be77043f485527f0a78d59038da13f0&__goaway_referer=https%3A%2F%2Fsearch.brave.com%2F)


``` ini
[Unit]
Description=A high performance web server and a reverse proxy server
Documentation=man:nginx(8)
After=network-online.target remote-fs.target nss-lookup.target
# Dont start nginx until the above are running

Wants=network-online.target
# Softer than Requires but nginx will launch after network-online target is reached but if that fails nginx will still start
 
ConditionFileIsExecutable=/usr/sbin/nginx
# Verify the files to run nginx are there

[Service]
Type=forking
PIDFile=/run/nginx.pid
# forking = nginx launches, spawns worker processes, then the original process exits 
# systemd tracks the workers via the PIDFile below instead of the original process

ExecStartPre=/usr/sbin/nginx -t -q -g 'daemon on; master_process on;'
# verifys nginx config syntex

ExecStart=/usr/sbin/nginx -g 'daemon on; master_process on;'
# Actual start command

ExecReload=/usr/sbin/nginx -g 'daemon on; master_process on;' -s reload
# Used by systemctl reload

ExecStop=-/sbin/start-stop-daemon --quiet --stop --retry QUIT/5 --pidfile /run/nginx.pid

TimeoutStopSec=5
# how long until the force kill is ran if the graceful stop does not work

KillMode=mixed
mixed = send SIGTERM to the main process, SIGKILL to all child processes

[Install]
WantedBy=multi-user.target
# multi-user target will run the nginx service when its loaded
```

## Edit a Unit File

> Don't edit a unit file directly since updates will overwrite this. Use the `systemctl edit` command to create an override file

```bash
sudo systemctl edit nginx.service
# This opens a blank file at: /etc/systemd/system/nginx.service.d/override.conf
# Add sections you want to change from the base unit file

sudo systemctl daemon-reload
# This will tell systemd to check over all its unit files again
```

Nuclear option to edit an entire unit file
```bash
sudo systemctl edit --full nginx.service
# This shows the entire unit file
# Files are wirrten to /etc/systemd/system
```

## Undo an Override

```bash
# Remove just the override 
sudo rm /etc/systemd/system/nginx.service.d/override.conf 

# Or revert a --full edit 
sudo systemctl revert nginx.service 

sudo systemctl daemon-reload
```

## Runtime Units
- Only this boot session. will be lost after reboot
	- Lives in RAM
- What uses Runtime
	- Snap/flatpack - runtime mount's for sandboxed apps
	- Containers - Docker, etc put runtime units here
```bash
systemd-run 
# Launch a one off service
	
systemctl --runtime 
# temporary unit change and the files will land under /run and not /etc
# Can be used in testing if you want to verify a change before commiting it

## EXAMPLE
# Run a backup script with memory and CPU constraints 
sudo systemd-run --unit=backup \ --property=MemoryMax=256M \ --property=CPUQuota=20% \ /usr/local/bin/backup.sh
```