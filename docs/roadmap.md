# Roadmap

Each pillar is built while studying the matching chapters of *How Linux Works*. Every change gets **before/after evidence** (command output or measurements).

## Build order: what each chapter unlocks

I only build what I've already learned. Each chapter ends with its project task.

| After chapter | Project task | Pillar |
|---|---|---|
| ✅ 1 The Big Picture | Architecture, node roles, first 2 incident reports | 8 |
| ✅ 2 Basic Commands | Folder layout, `feed` service user + `traders` group, permissions, audit who has sudo ([write-up](ch02-users-groups-permissions.md)) | 1 |
| 3 Devices | Map devices on both nodes; check the Pi's USB drive (INC-002) | 2 |
| 4 Disks & Filesystems | Data directory/volume, mount options, swap review, disk-full check | 2 |
| 5 Kernel Boot | Read boot logs, kernel parameters; prepare CPU isolation for tuning | 5 |
| 6 User Space Starts | **Fix INC-001**; feed handler as a proper systemd service | 3 |
| 7 Logging, Time, Jobs, Users | Log retention (INC-002), **chrony time sync**, scheduled reports with timers | 4, 7 |
| 8 Processes & Resources | Baseline latency; CPU pinning, priorities, cgroup limits | 5, 7 |
| 9 Network Config | Latency to exchanges, routing, DNS, firewall rules | 6, 1 |
| 10 Network Services | SSH hardening, fail2ban, health endpoint, packet capture | 1, 6 |
| 11 Shell Scripts | Deploy, health-check, and daily-report scripts | 8 |
| 12 File Transfer | rsync backups VM → Pi + restore drill; WireGuard tunnel | 8, 6 |
| 13 User Environments | Clean shell setup for an ops user | 1 |
| 15–16 Dev Tools & Compiling | Build a small C latency tool with `make` | 5 |
| 17 Virtualization | Containerise the feed handler; compare VM vs container | 8 |
| Beyond the book | Ansible to rebuild both nodes, Prometheus/Grafana, alerting | 7, 8 |

Chapter 14 (desktop & printing) has no project task: servers don't need it.

## 1. Access & security (Ch 2, 7, 10)
- [ ] Least-privilege service users and groups (no services running as root unless needed)
- [ ] File permissions on project, data, and log directories
- [ ] SSH: key-only login, password auth off, root login off
- [ ] sudo rules reviewed; audit which human accounts exist on each node
- [ ] Host firewall (ufw/nftables): default deny, allow only what's needed
- [ ] fail2ban for SSH

## 2. Storage (Ch 3, 4)
- [ ] Map devices and filesystems on both nodes (`lsblk`, `df`, `/etc/fstab`)
- [ ] Dedicated data directory/volume for market data, sensible mount options
- [ ] Disk-full alerting
- [ ] Pi USB drive health checks (INC-002)

## 3. Services & reliability (Ch 5, 6)
- [ ] Feed handler as a systemd service with its own user
- [ ] Sane restart policy: `Restart=on-failure` + start limits (lesson from INC-001)
- [ ] Dependencies (start after network and time sync)
- [ ] Service hardening options (`ProtectSystem`, `NoNewPrivileges`, ...)
- [ ] Health check endpoint/script

## 4. Time sync (Ch 7)
- [ ] chrony on both nodes; measure offset/jitter over time
- [ ] Alert if the clock drifts beyond a threshold
- [ ] Document why trading needs precise timestamps (regulation, ordering of events)
- [ ] Pi has no hardware clock (RTC): handle wrong time at early boot (seen in INC-002)

## 5. Low-latency tuning (Ch 5, 8, 9)
- [ ] Baseline latency measurements
- [ ] CPU pinning (`taskset`), priorities (`nice`/`chrt`), cgroup limits
- [ ] CPU frequency governor, IRQ affinity
- [ ] Kernel/network sysctl tuning, with before/after numbers

## 6. Networking (Ch 9, 10, 12)
- [ ] Measure latency/route to exchange endpoints (`ping`, `mtr`)
- [ ] DNS behaviour and caching
- [ ] WireGuard tunnel between the Pi and the VM
- [ ] Packet capture of the feed (`tcpdump`)

## 7. Observability (Ch 7, 8)
- [ ] Metrics + dashboards (netdata already on both; evaluate Prometheus/Grafana)
- [ ] Central logs, log rotation, retention
- [ ] Alerts: service down, disk full, clock drift, high latency

## 8. Ops, automation & incidents (Ch 11, 12, 17 + beyond)
- [ ] Shell scripts: deploy, health check, daily report
- [ ] rsync backups VM → Pi + **tested restore drill**
- [ ] Ansible playbooks to rebuild both nodes from scratch
- [ ] Containerise the feed handler (Docker/Podman) and compare
- [ ] Runbooks for common failures
- [x] Incident process: write a postmortem for every real problem
