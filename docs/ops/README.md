# bots.ac ops planning docs

Planning documents for the bots.ac server (manage.bots.ac, CapRover Pro on Ubuntu 24.04) and the bots.ac app deploy. Nothing here is a runbook to execute without Dani's approval.

This repo is public. IP addresses, the SSH port, host-specific port numbers, 1Password item IDs, tailnet names and local secret paths are replaced with placeholders like `<server-ip>` and `<ssh-port>`.

| File | What it is |
|---|---|
| [2026-10-16-premortem.md](2026-10-16-premortem.md) | Pre-mortem of all the plans below: failure stories, doc conflicts, assumptions, questions for Dani. Start here. |
| [2026-10-13-caprover-install-plan.md](2026-10-13-caprover-install-plan.md) | CapRover fresh-install plan. Phase 0 (host hardening) is done. |
| [2026-10-13-caprover-plan-security-audit.md](2026-10-13-caprover-plan-security-audit.md) | Security audit of the install plan. |
| [2026-10-13-server-health-audit.md](2026-10-13-server-health-audit.md) | Health and security audit of the server before Phase 0. |
| [2026-10-13-server-linux-doctor.txt](2026-10-13-server-linux-doctor.txt) | Raw Linux Doctor output. |
| [2026-10-13-server-inventory.txt](2026-10-13-server-inventory.txt) | Read-only inventory (listener and DNS sections removed). |
| [2026-10-16-operating-rules.md](2026-10-16-operating-rules.md) | Operating and security rules, accepted by Dani 2026-10-16. |
| [bots-ac-design.md](bots-ac-design.md) | bots.ac app design (Guilds.run fork). |
| [bots-ac-implementation-plan.md](bots-ac-implementation-plan.md) | bots.ac app implementation plan. |
