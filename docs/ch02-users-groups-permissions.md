# Chapter 2 task: users, groups and permissions on both nodes

**Pillar 1 (security / access control).** Built with what Chapter 2 of *How Linux Works* teaches: users, groups, file modes, `chmod`, `sudo`.

Status: ✅ done on 2026-11-14 on both nodes. Real before/after output is in the Evidence section.

## Design

The lab models a small trading firm with two boxes:

| Node | Role | Folders |
|---|---|---|
| `prod-vm` | **trading box**: runs the market-data feed | `/opt/trading/{bin,config,logs,data}` |
| `ops-pi` | **ops box**: log archive and reports | `/opt/trading/{archive,reports}` |

| Role | Account | On `prod-vm` | On `ops-pi` |
|---|---|---|---|
| Admin | the normal admin login | everything, through `sudo` (logged) | everything, through `sudo` (logged) |
| Service | `feed` (system user, no login) | runs the feed, writes logs and data | does not exist |
| Trader | `trader1`, group `traders` | reads logs only | reads reports only |

Rules:
- **Least privilege:** each account gets only what its job needs.
- **Same IDs on both boxes:** group `traders` = GID 2001, user `trader1` = UID 2001. Files and tar archives store numbers, not names.
- `trader1` is a practice account with no password and no SSH key: it can't log in from outside. It is tested with `sudo -u trader1`.
- The service user is `feed`; its group is renamed to **`feedgrp`** so that user and group names are never confused in `ls -l` output.
- No trading, no exchange keys, no orders (see `safety-rules.md`).

### Permissions on `prod-vm`

| Path | Owner:Group | Mode | Why |
|---|---|---|---|
| `/opt/trading` | root:root | 755 | everyone may pass through |
| `/opt/trading/bin` | root:root | 755 | the service can run its code but can't change it |
| `/opt/trading/config` | root:feedgrp | 750 | feed reads config; can't edit it; traders can't see it |
| `/opt/trading/logs` | feed:traders | 2750 | feed writes; traders read; setgid makes new files belong to `traders` |
| `/opt/trading/data` | feed:feedgrp | 700 | feed only |

### Permissions on `ops-pi`

| Path | Owner:Group | Mode | Why |
|---|---|---|---|
| `/opt/trading` | root:root | 755 | |
| `/opt/trading/archive` | root:root | 700 | admins only (raw log archives) |
| `/opt/trading/reports` | root:traders | 750 | traders may read reports |

## Runbook

### Step 0: before evidence (both nodes)
```bash
getent group traders; id feed; id trader1; ls -ld /opt/trading; ls -ld ~
```

### Step 1: group and users
`prod-vm`:
```bash
sudo groupadd --gid 2001 traders
sudo useradd --system --shell /usr/sbin/nologin --no-create-home feed
sudo useradd --uid 2001 --create-home --shell /bin/bash --groups traders trader1
sudo groupmod --new-name feedgrp feed      # give the service user's group its own name
id feed; id trader1; getent group traders feedgrp
```
`ops-pi`:
```bash
sudo groupadd --gid 2001 traders
sudo useradd --uid 2001 --create-home --shell /bin/bash --groups traders trader1
id trader1; getent group traders
```

### Step 2: folders and permissions
`prod-vm`:
```bash
sudo mkdir -p /opt/trading/{bin,config,logs,data}
sudo chown root:feedgrp /opt/trading/config
sudo chown feed:traders /opt/trading/logs
sudo chown feed:feedgrp /opt/trading/data
sudo chmod 755 /opt/trading /opt/trading/bin
sudo chmod 750 /opt/trading/config
sudo chmod 2750 /opt/trading/logs
sudo chmod 700 /opt/trading/data
ls -l /opt/trading
```
`ops-pi`:
```bash
sudo mkdir -p /opt/trading/{archive,reports}
sudo chown root:traders /opt/trading/reports
sudo chmod 755 /opt/trading
sudo chmod 700 /opt/trading/archive
sudo chmod 750 /opt/trading/reports
ls -l /opt/trading
```

### Step 3: test every role
`prod-vm` (expected result in the comment):
```bash
sudo -u feed touch /opt/trading/logs/feed.log        # works
sudo -u feed touch /opt/trading/config/hack          # Permission denied
sudo -u feed touch /opt/trading/bin/hack             # Permission denied
sudo -u trader1 ls -l /opt/trading/logs              # works
sudo -u trader1 touch /opt/trading/logs/fake.log     # Permission denied
sudo -u trader1 ls /opt/trading/config               # Permission denied
sudo -u trader1 ls /opt/trading/data                 # Permission denied
```
`ops-pi`:
```bash
sudo -u trader1 ls /opt/trading/reports              # works
sudo -u trader1 ls /opt/trading/archive              # Permission denied
```

### Step 4: fix the home-directory finding on `prod-vm`
Finding: the admin home directory was mode 755 (readable by every local user). Checked first: every service that uses it runs as the same user or as root, so 750 is safe.
```bash
chmod 750 ~
ls -ld ~
```

## Evidence

All output below is real, captured on 2026-11-14.

### Before
```
prod-vm and ops-pi:
  getent group traders        -> (nothing)
  id feed / id trader1        -> no such user
  ls -ld /opt/trading         -> No such file or directory
prod-vm:
  drwxr-xr-x 15 ubuntu ubuntu /home/ubuntu      <- 755: readable by every local user
```

### After: accounts
```
prod-vm$ id feed; id trader1; getent group traders
uid=996(feed) gid=996(feedgrp) groups=996(feedgrp)
uid=2001(trader1) gid=2002(trader1) groups=2002(trader1),2001(traders)
traders:x:2001:trader1

ops-pi$ id feed; id trader1
id: 'feed': no such user
uid=2001(trader1) gid=2002(trader1) groups=2002(trader1),2001(traders)
```
`trader1` and `traders` have the same numeric IDs on both nodes. `feed` exists only where the feed runs.

### After: folders
```
prod-vm$ ls -l /opt/trading
drwxr-xr-x 2 root root    4096 Nov 14 12:08 bin
drwxr-x--- 2 root feedgrp 4096 Nov 14 12:08 config
drwx------ 2 feed feedgrp 4096 Nov 14 12:08 data
drwxr-s--- 2 feed traders 4096 Nov 14 12:25 logs

ops-pi$ ls -l /opt/trading
drwx------ 2 root root    4096 Nov 14 23:24 archive
drwxr-x--- 2 root traders 4096 Nov 14 23:24 reports
```
(Timestamps differ because prod-vm runs on UTC and ops-pi on local time.)

### Role tests on prod-vm
| Test | Result | Pass? |
|---|---|---|
| `sudo -u feed touch /opt/trading/logs/feed.log` | created | ✅ feed can write logs |
| `sudo -u feed touch /opt/trading/config/hack` | `Permission denied` | ✅ feed can't change its config |
| `sudo -u feed touch /opt/trading/bin/hack` | `Permission denied` | ✅ feed can't change its code |
| `sudo -u trader1 ls -l /opt/trading/logs` | `-rw-r--r-- 1 feed traders 0 Nov 14 12:25 feed.log` | ✅ trader can read logs |
| `sudo -u trader1 touch /opt/trading/logs/fake.log` | `Permission denied` | ✅ trader can't write or fake logs |
| `sudo -u trader1 ls /opt/trading/config` | `Permission denied` | ✅ |
| `sudo -u trader1 ls /opt/trading/data` | `Permission denied` | ✅ |
| `sudo -u trader1 sudo -n true` | `sudo: a password is required` | ✅ trader has no sudo |

Note the log file's group: `feed` created it, but it belongs to `traders`. That is the setgid bit (`s` in `drwxr-s---`) on the logs directory doing its job.

### Role tests on ops-pi
| Test | Result | Pass? |
|---|---|---|
| `sudo -u trader1 ls /opt/trading/reports` | (empty, no error) | ✅ trader can read reports |
| `sudo -u trader1 ls /opt/trading/archive` | `Permission denied` | ✅ archive is admin-only |

### Home-directory fix on prod-vm
```
before: drwxr-xr-x 15 ubuntu ubuntu /home/ubuntu     (755)
$ chmod 750 ~
after:  drwxr-x--- 15 ubuntu ubuntu /home/ubuntu     (750)

$ sudo -u trader1 ls /home/ubuntu
ls: cannot open directory '/home/ubuntu': Permission denied
$ systemctl is-active portfolio-dashboard
active                                               <- existing service unaffected
```

## Sudo audit (read-only, 2026-11-14)

| | ops-pi | prod-vm |
|---|---|---|
| root account | locked | locked |
| members of group `sudo` | 1 admin | 2 admins |
| admin sudo rule | `NOPASSWD: ALL` | `NOPASSWD: ALL` |
| new accounts (`feed`, `trader1`) | no sudo | no sudo |

Follow-ups for later chapters:
- Review `NOPASSWD: ALL` for admins (convenient, but a stolen session is instant root).
- prod-vm has an unused account with an SSH key and a second admin account: review and disable what isn't needed (Ch7).
- The cloud provider's agent account may run several file commands as root: provider default, recorded here.

## What went wrong along the way (and what I learned)

- **Ran the prod-vm commands on ops-pi by mistake**, which created the `feed` user on the wrong box. Caught it from the prompt, reverted with `sudo userdel feed`, verified with `id feed`. Lesson: read the prompt before every command; know how to undo a change before making it.
- **User and group both named `feed`** made `ls -l` confusing. Renamed the group with `groupmod --new-name feedgrp feed`. No permissions had to be redone, because files store the numeric GID, not the name.
