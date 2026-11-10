# Security baseline (pillar 1)

Evidence of the starting point, before any hardening. Updated as chapters add tools.

## 2026-11-10: SSH exposure (found during Ch2 Lesson 2, `grep` on `/var/log/auth.log`)

| | prod-vm (public internet) | ops-pi (home LAN only) |
|---|---|---|
| Successful logins (current log) | 61, all mine, by SSH key | 269 (from my Mac) |
| Failed / invalid-user attempts | **847** from **61 unique IPs** in ~2.5 days | 0 |
| `PasswordAuthentication` | **no** (keys only) ✅ | **yes**: my Mac logs in with a password ⚠️ |
| `PermitRootLogin` | `without-password` (root by key allowed) ⚠️ | to check |
| Firewall | ufw **active** | to check |

```
$ grep -oE 'Invalid user [^ ]+' /var/log/auth.log | awk '{print $3}' | sort | uniq -c | sort -rn | head -5
    327 admin
    140 user
    124 user2
     13 test
     13 deploy
```

**Reading:** automated bots constantly scan public servers and guess common usernames. They all fail on prod-vm because password logins are disabled, so a guessed password is useless without a private key.

## Planned hardening (from roadmap pillar 1)
- [ ] prod-vm: `PermitRootLogin no`
- [ ] prod-vm: fail2ban to ban IPs after repeated failures
- [ ] ops-pi: switch my Mac to key login, then `PasswordAuthentication no`
- [ ] Review ufw rules on prod-vm; enable a firewall on ops-pi
- [ ] Review human accounts on prod-vm (4 exist: `opc`, `ubuntu`, `rabin`, `adminrabin`) and remove personal data from account comment (GECOS) fields
