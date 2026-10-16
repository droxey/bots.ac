# manage.bots.ac: health and security audit (2026-10-13)

Host: Ubuntu 24.04, kernel 6.8.0-139, 16 vCPU, 15 GiB RAM, no swap, disk 14G/464G (4%), up about 31 days. Public IP <server-ip>. Hostname is api.audiovau.lt.

## Tool used
- **Linux Doctor** v0.7.3: https://github.com/zShaD0w7x/linux-doctor (commit 4aa0506, 2026-10-10). The local skill `grok-bot-env-audit` names it.
- Vetting: Node CLI with zero runtime dependencies and no install scripts. Its checks are read-only. It changes the system only with `--fix --yes`, which I did not use. It makes network calls only on opt-in flags (`--push`, alerts, LLM). It writes only its history/cache under `~/.cache` and `~/.local/state`.
- How I ran it: nothing was installed system-wide. The repo and a portable Node 22.11 sit in `/home/dani/linux-doctor-tmp/`, because the server's own Node is 18 and the tool needs 20 or newer. I ran `sudo node doctor.js --plain` with read-only checks only. To remove it: `rm -rf ~/linux-doctor-tmp /tmp/ld.tgz`. Raw output: `2026-10-13-manage-bots-ac-linux-doctor.txt`.
- The security audit below uses manual read-only commands: sshd -T, ss, ufw/iptables/nft, pg_hba, and TCP probes from the box. No rules or services were changed.

## Overall verdict
**Fair as a machine (Linux Doctor score 61/100). Security posture is POOR.** The host has no firewall, and both Postgres and password SSH are reachable from the internet.

## Findings, ranked
### Critical / High
1. **No host firewall at all.** `ufw status`: inactive. iptables policy is INPUT/FORWARD ACCEPT, and the only chains are the ones Tailscale adds. Linux Doctor's "Firewall is active / risky services are firewalled" is a **false positive**: it counted the Tailscale chains. External probes also show no provider-side filtering, since closed ports answer *refused* rather than timing out.
2. **Postgres <pg-port> is open to the internet.** It listens on `0.0.0.0` and `::` (`listen_addresses='*'`), and the external probe gets OPEN. pg_hba allows `hostssl audiovault/audiovault_ci` from `0.0.0.0/0` and `::/0` with scram-sha-256. Exposure is limited to SSL plus password for one role, but it is still an internet-facing DB that can be brute-forced.
3. **SSH password authentication is effectively ON.** `sshd -T` gives `passwordauthentication yes`, because `/etc/ssh/sshd_config.d/50-cloud-init.conf` (yes) is read before `60-cloudimg-settings.conf` and the main file (no), and the first value wins. Linux Doctor's "password auth off" is a **false positive**. Port <ssh-port> is open publicly. The journal shows about 18.7k failed/invalid-user SSH lines in 7 days, and fail2ban is not installed.
4. **Failed unit `audiovault-health-check.service`.** The probe gets HTTP 503 from the Audio Vault API (exit 22, fires on a timer). The app or a dependency it checks is unhealthy.

### Medium
5. **Passwordless sudo for the whole `sudo` group** (`%sudo ALL=(ALL:ALL) NOPASSWD: ALL`, from cloud-init). Anyone who gets the dani account gets root instantly.
6. **The audiovault app listens on `*:<api-port>` and is publicly reachable**, bypassing nginx and TLS.
7. **Root has an authorized key** (an ed25519 key). `PermitRootLogin no` blocks it today. The dani account has 6 key lines: three named keys, plus a comment and blanks. Review that each one is still wanted.
8. **29 pending updates and a reboot required** (kernel 6.8.0-146 is installed but not running). unattended-upgrades is enabled, so the backlog is non-security packages and the kernel.
9. **Journal is 2.3 GB** with no cap.
10. **X11Forwarding yes** and MaxAuthTries 6: minor hardening gaps.

### Low / Info
- No swap. That is fine at 12 GiB available, but a small swapfile adds headroom for Docker builds.
- No SMART data (VM), no fwupd, no disk encryption: normal for a VPS.
- Backups: Linux Doctor reports "no backup tool". However, a root cron job dumps Postgres nightly to `/var/backups/audiovault` with 7-day retention, which is **on the same disk**, so there is no offsite copy.
- Good: AppArmor enforcing, NTP synced, TLS certs have more than 30 days left, apt healthy, no unusual SUID binaries outside /usr, no world-writable files in /etc or /usr/local, OOM and filesystem errors clean.
- Tailscale: up as `<server-tailnet-name>` (<server-tailnet-ip>). It only touches its own chains. One peer, `<offline-peer>`, has been offline 14 days and could be pruned.

## Security audit: external reachability (TCP probes from the box to <server-ip>)
| Port | Result | Listener |
|---|---|---|
| 22 | refused | none (SSH is on <ssh-port>) |
| 80, 443 | OPEN | nginx |
| <pg-port> | **OPEN** | postgres |
| <api-port> | **OPEN** | audiovault |
| <ssh-port> | OPEN | sshd |
| <setup-port>, <registry-port>, <swarm-mgmt-port>, <swarm-gossip-port> | refused, about 25 ms | none |

Every closed port answers *refused* quickly instead of timing out, so **neither Servarica nor the host filters anything**: every port is reachable once something listens on it. UDP 443, <swarm-gossip-port> and <vxlan-port> could not be tested reliably (nc/nmap are not on the box, and UDP has no refused signal without ICMP). With no host firewall, though, they will be open once something binds to them.

## CapRover port readiness
Port requirements come from https://caprover.com/docs/firewall.html:
- Public: 80/tcp, 443/tcp, 443/udp (HTTP/3), and <setup-port>/tcp for setup only (close it after a domain is attached). <registry-port>/tcp is the registry, to be exposed only if an external client needs it.
- Cluster only, between trusted nodes: <swarm-mgmt-port>/tcp, <swarm-gossip-port>/tcp+udp, <vxlan-port>/udp. The docs say not to expose <vxlan-port> publicly.

Readiness:
- **Reachability:** every port is reachable today. Nothing blocks <setup-port>, <registry-port>, <swarm-mgmt-port>, <swarm-gossip-port> or <vxlan-port>. That is a problem rather than a plus: on a single node, <swarm-mgmt-port>, <swarm-gossip-port> and <vxlan-port> should be **closed** to the public.
- **Conflict:** **nginx already holds 80/443.** CapRover's own nginx needs those ports, so the current nginx sites must be migrated into CapRover or moved first.
- **Docker bypasses UFW.** Ports Docker publishes skip UFW rules. Enabling UFW later will not protect CapRover-published ports; that takes DOCKER-USER chain rules.
- **Tailscale and iptables:** past notes mention an iptables-legacy vs nft issue with tailscaled. Docker on 24.04 uses iptables-nft, so confirm Tailscale and Docker agree on the backend.
- Postgres on the host must stay reachable from containers, through the docker0/overlay gateway, without staying open to the world.

## Recommended fixes (not applied; need Dani's approval)
1. Make SSH key-only: in `50-cloud-init.conf` set `PasswordAuthentication no` (or delete the file), run `sshd -t && systemctl reload ssh`, and keep a second session open while testing. Optionally set `X11Forwarding no` and `MaxAuthTries 3`, and install fail2ban.
2. Turn on a default-deny firewall. **Add the allow rules before enabling it**: `ufw allow <ssh-port>/tcp`, then `ufw allow 80,443/tcp`, `ufw allow 443/udp`, `ufw allow in on tailscale0`, then `ufw enable`. Allow <setup-port> only temporarily during CapRover setup. Keep <swarm-mgmt-port>, <swarm-gossip-port> and <vxlan-port> closed publicly.
3. Postgres: set `listen_addresses = 'localhost,<server-tailnet-ip>'` (plus the Docker bridge IP once CapRover exists). Swap the `0.0.0.0/0` and `::/0` hostssl lines for specific CI IPs or the tailnet. If CI needs public access, firewall it to known source IPs.
4. Bind audiovault to 127.0.0.1:<api-port> behind nginx, or block <api-port> at the firewall.
5. Fix the API 503 behind `audiovault-health-check`.
6. Apply updates and reboot in a maintenance window. Set the journald cap `SystemMaxUse=500M`.
7. Require a password for sudo (`%sudo ALL=(ALL:ALL) ALL`), or limit NOPASSWD to specific commands. Remove the root authorized_keys entry if it is unused.
8. Copy the Postgres dumps offsite (R2, restic, etc.).
9. Before CapRover: plan the nginx-to-CapRover migration for 80/443, add DOCKER-USER rules, and settle the iptables backend.
