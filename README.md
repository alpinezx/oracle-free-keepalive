# Oracle Always Free Keepalive: Setup & Reference Guide

A systemd-backed script that keeps an Oracle Always Free instance active by
running short CPU bursts on a schedule, so it doesn't get reclaimed for
sitting idle. Runs natively on the VM (not Docker), and survives reboots,
including a daily scheduled reboot.

Repo: https://github.com/alpinezx/oracle-free-keepalive

---

## Why this exists

Oracle reclaims Always Free instances that sit idle for 7 straight days,
based on average CPU utilization and network traffic staying below 20%.
This script keeps the instance lightly active so that never happens.

---

## What it actually does

- Runs a 5-minute CPU burst (`stress-ng`) every 4 hours
- Pings a lightweight URL alongside each burst for network activity
- Sleeps completely idle between bursts (roughly 2% total load over a day)
- Runs as a native `systemd` service, so it starts on boot automatically

Files it creates on the VM:

| Path | Purpose |
|---|---|
| `/opt/keepalive/keepalive-worker.sh` | The actual burst loop |
| `/etc/systemd/system/keepalive.service` | systemd unit definition |
| `/var/log/keepalive.log` | Log of burst cycles |

---

## First-time setup

Clone the repo directly onto the instance:

```bash
git clone https://github.com/alpinezx/oracle-free-keepalive.git
cd oracle-free-keepalive
chmod +x keepalive.sh
sudo bash keepalive.sh
```

Running it with no arguments opens the interactive menu. From there:

1. Choose **1) Install**
2. If `stress-ng` isn't already installed, it will ask:
   `Install it now via apt? [Y/n]` — press Enter or type `Y`
3. It writes the worker script, creates the systemd service, enables it at
   boot, and starts it immediately

That's it. No further setup needed.

### Updating later

If the script gets updated in the repo, pull the latest version and
reinstall:

```bash
cd oracle-free-keepalive
git pull
sudo bash keepalive.sh install
```

Reinstalling is safe to run again, it just rewrites the worker script and
systemd unit and restarts the service.

---

## The menu

| Option | What it does |
|---|---|
| 1) Install | First-time setup (also usable to repair/reinstall) |
| 2) Start | Resume now |
| 3) Stop | Pause now, but it **will** restart on next reboot |
| 4) Enable | Resume + survive reboots |
| 5) Disable | Stop **and** stay off through reboots |
| 6) Status | Full health check: summary, systemd detail, last 10 log lines |
| 7) Uninstall | Fully removes the service, unit file, and installed files |
| 8) Check/install stress-ng | Verify or install the CPU tool independently |
| 9) Exit | Quits the menu |

Run it any time with:

```bash
sudo bash keepalive.sh
```

---

## Command-line shortcuts (no menu)

Same actions, callable directly, useful for scripting or quick one-liners:

```bash
sudo bash keepalive.sh install
sudo bash keepalive.sh start
sudo bash keepalive.sh stop
sudo bash keepalive.sh enable
sudo bash keepalive.sh disable
sudo bash keepalive.sh status
sudo bash keepalive.sh uninstall
```

---

## "I want to actually use the server for something"

Pause it, do your work, resume when done:

```bash
sudo bash keepalive.sh stop      # pauses now, comes back on next reboot regardless
sudo bash keepalive.sh disable   # pauses now AND stays off through reboots too
```

If you're just doing a quick task, `stop` is enough since real usage on the
box generates its own CPU/network activity anyway. Use `disable` if you know
you won't be touching it again for a while and want it to stay quiet even
through a reboot.

To bring it back:

```bash
sudo bash keepalive.sh start     # if it was just stopped
sudo bash keepalive.sh enable    # if it was disabled
```

---

## Monitoring it

**Watch it live:**
```bash
tail -f /var/log/keepalive.log
```
`Ctrl+C` to stop watching (doesn't affect the service).

**Check CPU time is climbing:**
Menu → option 6, look at the `CPU:` line, check again after a minute.

**Watch it happen in real time:**
```bash
htop
```
Look for `stress-ng-cpu` in the process list during a burst window.

**The one that actually matters long-term:**
OCI Console → Compute → Instances → your instance → Metrics → CPU
Utilization. This is what Oracle itself uses to decide reclamation.

---

## Adjusting the schedule

Edit the worker script directly:

```bash
sudo nano /opt/keepalive/keepalive-worker.sh
```

Change these two lines near the top:

```bash
CPU_BURST_SECONDS=300      # how long each burst lasts, in seconds
INTERVAL_HOURS=4           # how often it runs
```

Then apply the change:

```bash
sudo systemctl restart keepalive
```

---

## Is 100% CPU for 5 minutes safe?

Yes. A few reasons:

- It's a virtual CPU on shared cloud infrastructure, not physical hardware
  you own
- Modern CPUs are designed to run at full load indefinitely
- 5 minutes every 4 hours is roughly 2% of total time
- `stress-ng` is a purpose-built, widely used tool designed exactly for this

## Will Oracle flag this as abuse?

No. Always Free only reclaims *underused* resources, there's no penalty for
using your own allocated CPU fully. This would only be a concern if it were
doing something like crypto mining or generating botnet-like network
traffic, neither of which applies here.

---

## Quick troubleshooting

**Status says "Not installed yet"**
→ Run option 1 (Install).

**Status says "Stopped (survives reboot)"**
→ That's fine if you paused it deliberately. Run option 2 (Start) to resume.

**Status says "will NOT survive reboot"**
→ Run option 4 (Enable).

**Want to verify `stress-ng` is actually being used (not the fallback loop)**
→ Run option 6 (Status) during a burst, check the `CGroup:` section. You
should see `stress-ng --cpu 1 --timeout 300s`, not `bash -c "while true"`.

**Something seems broken and you want a clean slate**
```bash
sudo bash keepalive.sh uninstall
sudo bash keepalive.sh install
```
