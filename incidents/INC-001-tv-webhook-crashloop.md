# INC-001: Retired service crash-looping on prod-vm

| | |
|---|---|
| Status | 🔴 **Open**, fix planned in Chapter 6 (systemd) |
| Node | prod-vm |
| Detected | 2026-11-09, during Chapter 1 Lesson 3 lab (`systemctl is-active` showed `activating`) |
| Severity | Low: no functional impact; wasted CPU and log noise |

## Summary
`tv-webhook.service`, a leftover from a retired trading bot, is still **enabled** on prod-vm, but the program it starts was deleted. It fails instantly with `status=203/EXEC`, and `Restart=always` makes systemd restart it every ~15 seconds, forever.

## Impact
- **5,579 restarts** counted at 2026-11-09 12:32 UTC (counter reset by the VM reboot on 2026-11-08 12:54 UTC, so about 1 restart per 15 s).
- Journal filled with failure messages; small CPU waste.
- Would hide real problems: a noisy, always-failing unit trains people to ignore alerts.

## Evidence
```
$ systemctl status tv-webhook
● tv-webhook.service - TradingView manual-signal webhook receiver (queues signals only; no order placement)
     Loaded: loaded (/etc/systemd/system/tv-webhook.service; enabled; preset: enabled)
     Active: activating (auto-restart) (Result: exit-code)
    Process: ExecStart=/home/ubuntu/Crypto/.venv/bin/python3 /home/ubuntu/Crypto/manual_signal_webhook.py (code=exited, status=203/EXEC)

$ systemctl show tv-webhook -p Restart -p NRestarts
Restart=always
NRestarts=5579

$ ls /home/ubuntu/Crypto/.venv/bin/python3
ls: cannot access '/home/ubuntu/Crypto/.venv/bin/python3': No such file or directory
```
- `203/EXEC` = systemd could not **exec()** the program (Chapter 1: fork + exec). The file doesn't exist.
- The journal for this unit goes back to 2026-08-11.
- On ops-pi the same service still runs normally (its venv exists) and listens only on `127.0.0.1:8800`.

## Root cause (preliminary)
1. The bot was retired and its Python virtualenv deleted, **but the systemd unit was left enabled**.
2. `Restart=always` with a 15 s delay never hits systemd's start limit (default 5 starts per 10 s), so the loop never stops.
3. Nothing alerted on a unit stuck restarting.

## Resolution
*Pending, Chapter 6.* Plan: `systemctl disable --now tv-webhook`, archive the unit file, `daemon-reload`, confirm with `systemctl status` and `systemctl --failed`.

## Action items
- [ ] Stop and disable the unit on prod-vm; decide whether to keep it on ops-pi
- [ ] Decommissioning checklist: when retiring an app, disable its units, timers, and cron jobs **first**
- [ ] Use `Restart=on-failure` with `StartLimitIntervalSec`/`StartLimitBurst` for project services
- [ ] Alert on units with high `NRestarts` or in a failed state (Observability pillar)

## Lessons
- A process the kernel can't `exec()` fails instantly. The exit code tells you where to look.
- Retiring software is a change too, and needs a checklist.
