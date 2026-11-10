# Trading Infrastructure Lab

A two-node lab where I build, secure, tune, monitor and operate **trading-style Linux infrastructure** while studying *How Linux Works* (3rd ed.). Study notes are in [linux-homelab](https://github.com/rbkunwar/linux-homelab).

**Focus: infrastructure.** The workload is a small public market-data feed handler. It exists only so there's something real to keep alive, fast and observable. **No trading, no exchange API keys, no orders, no money.**

## Architecture

```
┌──────────────────── VM: "production server" (cloud) ────────────────────┐
│  workload: small market-data feed handler (public data, no trading)      │
│  ── hardened · time-synced · latency-tuned · monitored · backed up ──    │
└───────────────▲──────────────────────────────────────────┬──────────────┘
                │  WireGuard VPN tunnel (private link)      │ metrics, logs, backups
┌───────────────┴──────── Raspberry Pi: "ops / control box" ▼──────────────┐
│  config management (Ansible) · monitoring + alerts · backup store ·       │
│  runbooks · incident reports · deploy scripts                             │
└───────────────────────────────────────────────────────────────────────────┘
```

| Node | Role | Hardware | OS |
|---|---|---|---|
| `prod-vm` | production server | cloud VM, 2 vCPU x86_64, 1 GB RAM | Ubuntu 24.04 |
| `ops-pi` | ops / control box | Raspberry Pi 4, 4 cores ARM64, 4 GB RAM, USB boot | Ubuntu 26.04 |

## Pillars

| # | Pillar | Status |
|---|---|---|
| 1 | Access & security | 🟡 baseline captured ([docs/security-baseline.md](docs/security-baseline.md)) |
| 2 | Storage | ⏳ planned |
| 3 | Services & reliability | ⏳ planned |
| 4 | Time sync | ⏳ planned |
| 5 | Low-latency tuning | ⏳ planned |
| 6 | Networking | ⏳ planned |
| 7 | Observability | ⏳ planned |
| 8 | Ops, automation & incidents | 🟡 2 incidents open |

Details and the chapter mapping: **[docs/roadmap.md](docs/roadmap.md)**

## Incidents

| ID | Title | Node | Status |
|---|---|---|---|
| [INC-001](incidents/INC-001-tv-webhook-crashloop.md) | Retired service crash-looping (5,500+ restarts) | prod-vm | 🔴 open |
| [INC-002](incidents/INC-002-pi-unexpected-restarts.md) | Pi freezes / unexpected restarts | ops-pi | 🟠 investigating |

## Rules
See [docs/safety-rules.md](docs/safety-rules.md).
