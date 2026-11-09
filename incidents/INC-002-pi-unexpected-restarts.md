# INC-002: ops-pi freezes / unexpected restarts

| | |
|---|---|
| Status | 🟠 **Investigating** (Chapters 3–4 storage, 7 logging) |
| Node | ops-pi |
| Reported | Before the lab started: "frozen or lost power a few times" |
| Suspects | Power supply, USB boot drive |

## Summary
The Pi has frozen or lost power several times. The latest unexpected restart is confirmed from the journal: the previous boot **stopped abruptly with no shutdown sequence**.

## Evidence (collected 2026-11-10)
```
$ journalctl --list-boots
IDX BOOT ID                          FIRST ENTRY                  LAST ENTRY
 -1 f30bc640...                      Thu 2026-11-05 22:38:45 AEDT Sat 2026-11-07 21:42:39 AEDT
  0 db9d2a31...                      Tue 2026-07-28 06:44:02 AEST Tue 2026-11-10 00:00:03 AEDT

$ journalctl -b -1 -n 4      # last lines before it died: ordinary SSH logouts, then silence
2026-11-07T21:42:39 sshd-session: Disconnected from user rabindrakunwar ...
2026-11-07T21:42:39 systemd-logind: Session 174 logged out. Waiting for processes to exit.

$ uptime -s
2026-11-07 21:51:04          # back up ~8.5 minutes later
```
- **No system shutdown messages** at the end of boot -1 (the only `shutdown.target` lines belong to per-user systemd instances earlier that night), so this was **not** a clean reboot or shutdown.
- **No out-of-memory kills** in boot -1, so it's unlikely memory exhaustion killed it.
- Hardware watchdog is active: `systemd: Using hardware watchdog /dev/watchdog0: 'Broadcom BCM2835 Watchdog timer'`.
- No under-voltage/throttling messages in the current boot's kernel log (`dmesg`). `vcgencmd get_throttled` currently fails (`Can't open device file: /dev/vcio`), so it needs fixing to read the firmware's throttle flags.
- The journal only holds **2 boots**, so history of earlier freezes is lost.

### Side finding: wrong clock at boot
Boot 0's first entry is dated **2026-07-28** although it really started on 2026-11-07. The Pi has **no battery-backed hardware clock (RTC)**, so it boots with an old saved time until NTP corrects it. Early-boot log timestamps are therefore wrong. This feeds the Time sync pillar.

## Hypotheses
| Hypothesis | For | Against / to check |
|---|---|---|
| Power supply dips (under-voltage) | Classic Pi failure; "lost power" reported | Need throttle flags (`get_throttled`) |
| USB boot drive hangs / errors | Suspect hardware; USB storage can stall I/O | Check `dmesg` for I/O errors, SMART data, `wa` in top |
| Kernel hang caught by watchdog | Watchdog is active | 8.5 min gap is long for a watchdog reset; journal may not be flushed |
| Real power loss (outage/unplugged) | Gap fits a person restoring power | Ask: was anything unplugged? |

## Next steps
- [ ] Make throttle/under-voltage readable (`vcgencmd` permissions, or `/sys` throttle file)
- [ ] Check the USB drive: `dmesg` I/O errors, `smartctl` (if supported), filesystem check plan
- [ ] Keep more journal history (persistent storage + size limits) so the next freeze leaves evidence
- [ ] Monitor voltage/throttling, temperature, and disk latency continuously (Observability pillar)
- [ ] Try a known-good official power supply

## Lessons (so far)
- An abrupt end to a boot's journal with no shutdown messages = crash or power loss.
- Without enough log retention and monitoring, you can't diagnose the past. Instrument *before* the next failure.
