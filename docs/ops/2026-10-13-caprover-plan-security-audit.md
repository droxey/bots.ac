# Security audit: CapRover fresh-install plan for manage.bots.ac

Date: 2026-10-13. Scope: `/workspace/reports/2026-10-13-caprover-fresh-install-plan.md`, the inventory, and read-only checks on the server and box. Nothing was changed.

Sources: https://caprover.com/docs/get-started.html, https://caprover.com/docs/firewall.html, https://caprover.com/docs/troubleshooting.html, https://docs.docker.com/engine/network/packet-filtering-firewalls/, https://docs.docker.com/engine/swarm/swarm-tutorial/#open-protocols-and-ports-between-the-hosts, https://letsencrypt.org/docs/rate-limits/, https://securitylab.github.com/resources/github-actions-preventing-pwn-requests/, https://docs.github.com/en/actions/security-for-github-actions/security-guides/security-hardening-for-github-actions, https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry, caprover CLI 2.4.4 source (`built/commands/serversetup.js`).

## Server facts checked today

- sshd: `port <ssh-port>`, `permitrootlogin no`, `pubkeyauthentication yes`, but **`passwordauthentication yes`** and **`kbdinteractiveauthentication yes`**. fail2ban is not running.
- Postgres: `listen_addresses = *`. pg_hba has `hostssl audiovault audiovault 0.0.0.0/0` and `::/0`, plus the same for `audiovault_ci` (scram-sha-256).
- iptables: every policy is ACCEPT, and the only rules are Tailscale's `ts-input`/`ts-forward` chains. ufw is inactive. unattended-upgrades is active.
- Box: `<op-token-file>` is mode 600, owner `box`. Every agent on the shared box runs as `box`.
- CLI 2.4.4 `serversetup` calls `forceSsl(true)` before `changePass(...)`, so the new password is sent only after HTTPS is forced.

## Critical

### C1. Anyone on the internet can take over between install and serversetup (captain42)
- Evidence: Task 2 Step 3 publishes `-p <setup-port>:<setup-port>` on the public IP with the default password `captain42`. Task 3 runs later, from the box. There is no ufw and every iptables policy is ACCEPT. Docker-published ports bypass host firewall rules anyway (Docker packet-filtering docs).
- Risk: The dashboard controls `/var/run/docker.sock`, so dashboard access equals root on the host. That puts Postgres data, audiovault-api and Tailscale identity at risk. Bots scan for open CapRover dashboards.
- Fix: Before the installer runs, insert `sudo iptables -I DOCKER-USER -p tcp --dport <setup-port> ! -s <box egress IP> -j DROP` (get the box egress IP at run time with `curl -s https://ifconfig.me`). Run Task 2 Step 3 and Task 3 Step 2 back to back in one gated step with no pause between them. Afterwards, verify `captain42` is rejected.

### C2. PR previews have no guardrails against untrusted code
- Evidence: Task 7 lists only the wildcard and the 2FA question.
- Risk: A workflow on `pull_request_target`, or any preview job that runs fork code with access to secrets, leaks deploy credentials (the "pwn request" pattern, GitHub Security Lab). Creating and deleting apps needs dashboard-level (root-equivalent) credentials in CI. A preview's Dockerfile can run anything on the host's Docker and burn CPU and RAM (the host has 15 GiB and no swap) next to Postgres.
- Fix: Add these to Task 7 as rules:
  - Trigger only on `pull_request` from same-repo branches (`if: github.event.pull_request.head.repo.full_name == github.repository`). Never use `pull_request_target`.
  - Keep the admin credential in a GitHub Environment `previews` with required reviewers.
  - Set memory and CPU limits per preview app.
  - Delete the app when the PR closes, plus a daily sweep that deletes previews older than 7 days.
  - Cap concurrent previews at 3.
  - Do not ship Task 7 until a test shows the CLI can authenticate with 2FA on (`CAPROVER_OTP_TOKEN`).

## High

### H1. SSH accepts passwords
- Evidence: `passwordauthentication yes`, `kbdinteractiveauthentication yes`, no fail2ban. The plan makes the host a higher-value target.
- Fix (new Task 0 step, gated): write `/etc/ssh/sshd_config.d/10-hardening.conf` with `PasswordAuthentication no`, `KbdInteractiveAuthentication no` and `MaxAuthTries 3`. Run `sudo sshd -t`, then `sudo systemctl reload ssh`. Verify from a **second** session (`ssh -o PreferredAuthentications=password manage.bots.ac` must fail, and key login must work) before closing the first. Tailscale SSH is the fallback path.

### H2. Swarm ports open to the internet
- Evidence: `docker swarm init` (run by the installer) listens on <swarm-mgmt-port>/tcp and <swarm-gossip-port>/tcp+udp, and <vxlan-port>/udp carries VXLAN. CapRover's firewall page says to allow these only between trusted nodes and that exposing <vxlan-port> publicly can open the overlay to attack. The plan's Global constraints list no rule for them.
- Fix: Add host INPUT drops after install. These are INPUT rules, so they work even with ufw off and don't touch <ssh-port> or `tailscale0`:
  `sudo iptables -A INPUT -p tcp -m multiport --dports <swarm-mgmt-port>,<swarm-gossip-port> ! -i lo -j DROP; sudo iptables -A INPUT -p udp -m multiport --dports <swarm-gossip-port>,<vxlan-port> ! -i lo -j DROP`.
  Persist them with a systemd oneshot (see M3). Don't enable the self-hosted registry (<registry-port>); GHCR replaces it. Verify with `nc -zv -w5 <server-ip> <swarm-mgmt-port>` from the box: it must time out.

### H3. The dashboard is root-equivalent and gets 2FA too late
- Evidence: Pro and 2FA come in Task 4, after DNS, HTTPS and setup. `captain.dev.bots.ac` is public and guessable, and the root domain shows up in Certificate Transparency logs once the certificate is issued.
- Fix: Do Task 4 right after Task 3, with no other work in between. Turn on login alerts before 2FA. Optionally restrict the dashboard to the box IP and the tailnet with CapRover's custom nginx config for the captain dashboard (Settings → NGINX config). Then 2FA protects a login page that is already private.

### H4. Postgres is public, and the plan's own Task 0 dump crosses the network
- Evidence: pg_hba allows `audiovault` from `0.0.0.0/0` over TLS. Docker doesn't change this, because host ports aren't Docker-published. Containers can reach the host at `172.17.0.1:<pg-port>`, but pg_hba rejects 172.16/12. The plan defers this to a follow-up.
- Fix: Keep it as a separate follow-up, but make it the first one. Remove the `0.0.0.0/0` and `::/0` lines and keep `100.64.0.0/10` (tailnet). If CI needs `audiovault_ci`, route it over Tailscale. In this plan, add a Task 2 check that `pg_hba` was not changed (`sudo md5sum /etc/postgresql/16/main/pg_hba.conf` before and after).

## Medium

### M1. Secret handling around `op read` and serversetup
- Evidence: Task 3 Step 2 runs `export OP_SERVICE_ACCOUNT_TOKEN=...` and `trap ... EXIT` in an interactive shell. The password goes over stdin and through a 600-mode temp file, so it never shows up in process args (good). But:
  1. In a persistent shell the token stays exported and the trap stays set until the shell exits.
  2. Any agent on the shared box can read the token file.
  3. After login, the CLI saves an auth token in the `configstore` file (`~/.config/configstore/caprover.json`).
  4. `/tmp` is shared on the box.
- Fix:
  1. Run the whole block in a subshell `( ... )` and drop `export`: `OP_SERVICE_ACCOUNT_TOKEN="$(cat ...)" op read ...`.
  2. Use `mktemp -p "$XDG_RUNTIME_DIR"` or a 700 dir under the agent's own directory.
  3. Limit the 1Password service account to read-only on vault Nebula.
  4. Run `caprover logout -n manage-bots-ac` afterwards, or delete the configstore entry.
  5. Check the box shell history settings (HISTFILE is empty today, which is good) and never use `set -x`.

### M2. Deploy pipeline supply chain
- Evidence: The workflow pins actions by tag (`actions/checkout@v4`, `docker/login-action@v3`, `docker/build-push-action@v6`, `caprover/deploy-from-github@v2.0.0`). The v2.0.0 inputs haven't been checked. `CAPROVER_APP_TOKEN` is a plain repo secret, so any workflow on any branch can use it.
- Fix: Pin every action to a full commit SHA with the tag as a comment. Move `CAPROVER_APP_TOKEN` into an Environment `production` limited to `main`. Read the action's `action.yml` at that SHA before use. Keep `permissions` at job level, as it is now.

### M3. Firewall rules won't survive a reboot, and the persistence tool is risky
- Evidence: The plan uses `netfilter-persistent save`, which isn't installed. Saving the full ruleset also captures Docker's and Tailscale's dynamic chains and restores stale copies at boot.
- Fix: Use a systemd oneshot `fw-caprover.service` (`After=docker.service tailscaled.service`) that inserts only the DOCKER-USER rule for <setup-port> and the INPUT drops from H2. It must first insert `-A INPUT -p tcp --dport <ssh-port> -j ACCEPT` and `-A INPUT -i tailscale0 -j ACCEPT`. Verify after a reboot that SSH on <ssh-port> and `tailscale status` both work.

### M4. Closing port <setup-port> with `--publish-rm` may not stick
- Evidence: CapRover manages `captain-captain` and can re-apply its own spec on restart or update. The plan already names the iptables fallback.
- Fix: Make the DOCKER-USER rule (C1) the main control and leave it in place permanently. Treat `--publish-rm` as optional. Recheck after every CapRover update.

### M5. GHCR token scope and lifetime
- Evidence: A classic PAT with `read:packages` can read every package the `droxey` account can see, including org packages. GHCR doesn't accept fine-grained PATs.
- Fix: Use a dedicated machine user or org bot account with access only to the needed packages. Set a 90-day expiry with a rotation reminder, and keep it in 1Password next to the admin item. It must never get `write:packages` or `repo`.

### M6. Backups are unencrypted on the shared box
- Evidence: Task 0 writes `pg_dumpall.sql.gz` and `/etc/letsencrypt` (private keys) to `/workspace/backups`. Task 6 adds `/captain/data`, which holds the registry PAT, the hashed admin password and certificate keys. Every agent can read them.
- Fix: Pipe through `age -r <Dani's age public key>` (or `gpg -e`) before writing, and keep the directory at mode 700. Test a restore of `/captain/data` once. Add an off-box copy (e.g. R2) for the encrypted files.

## Low

### L1. Image pinning is by tag, not digest
`caprover/caprover:1.15.4` is a mutable tag. Pin `caprover/caprover:1.15.4@sha256:<digest>`, using the digest from `docker buildx imagetools inspect caprover/caprover:1.15.4` at run time. Do the same for the app images: deploy `ghcr.io/...@sha256` (the build-push-action output `digest`) rather than `:sha` tags.

### L2. Docker apt key isn't fingerprint-checked
Add `gpg --show-keys /etc/apt/keyrings/docker.asc | grep -q 9DC858229FC7DD38854AE2D88D81803C0EBFCD88` after the download.

### L3. Unattended Docker upgrades
unattended-upgrades only covers the Ubuntu security origins by default, so Docker CE from download.docker.com won't auto-upgrade. Run `sudo apt-mark hold docker-ce docker-ce-cli containerd.io` and upgrade on purpose alongside CapRover updates.

### L4. Wildcard DNS and Let's Encrypt
The `*.dev.bots.ac` wildcard points at the server. A subdomain takeover is only possible if the IP is released while the record remains, so add "delete `*.dev.bots.ac` before decommissioning" to the rollback. Let's Encrypt limits are 50 certificates per registered domain (`bots.ac`) per 7 days and 5 duplicates per exact name per week. Previews issue one certificate per app over HTTP-01, so the C2 cap keeps them under the limit. Preview hostnames appear in CT logs, so don't put secrets or customer names in branch-derived names.

### L5. Rollback gaps
- Purging Docker leaves its iptables chains until reboot.
- Disabling `certbot.timer` lets the `api.audiovau.lt` certificate expire, which is accepted by decision.
- Add explicit rollback items: remove the `*.dev.bots.ac` record, run `sudo iptables -D` for each added rule (or disable `fw-caprover.service`), and revoke the GHCR PAT and app tokens.

## Plan changes to make

1. Task 2 Step 3 + Task 3 Step 2: add the DOCKER-USER rule for <setup-port> before install, then run install and serversetup in one gated step (C1, M4).
2. Move Task 4 (Pro, login alerts, 2FA) to right after Task 3. Optionally add a box/tailnet allowlist for captain (H3).
3. New firewall task: `fw-caprover.service` with the <ssh-port> and tailscale0 accepts first, then drops for <swarm-mgmt-port>/<swarm-gossip-port>/<vxlan-port>, and the <setup-port> rule. Verify with nc and a reboot (H2, M3).
4. New gated Task 0 step: SSH key-only hardening with a second-session check (H1).
5. Task 3 Step 2: subshell, no `export`, private temp dir, `caprover logout` afterwards, read-only service account (M1).
6. Task 5: SHA-pin actions, Environment-scoped app token, deploy by digest, dedicated GHCR reader account with an expiry (M2, M5, L1).
7. Task 6 and Task 0: encrypt backups with age and add an off-box copy (M6).
8. Task 7: same-repo PRs only, no `pull_request_target`, a reviewer-gated Environment, resource limits, cleanup on close plus a TTL sweep, a cap of 3, and a proven 2FA CLI login as an entry criterion (C2, L4).
9. Pin the CapRover image digest and the Docker key fingerprint, and hold the Docker packages (L1-L3).
10. Follow-up after this plan: close public Postgres (H4).
