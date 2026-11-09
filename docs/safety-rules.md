# Safety rules

1. **Infrastructure only.** No trading, no orders, no exchange API keys, no real money. Market data is public.
2. **No secrets in Git:** no private keys, tokens, passwords, or `.env` files.
3. **No IP addresses or personal email in Git.** Nodes are referred to by role (`prod-vm`, `ops-pi`).
4. **Read the prompt before risky commands** (`reboot`, `rm`, `systemctl stop`). Confirm with `hostname`.
5. **Evidence before and after every change.** Save the command output in the relevant doc.
6. **Every real problem gets a postmortem** in `incidents/`.
7. Treat `prod-vm` like production: plan changes, have a rollback.
