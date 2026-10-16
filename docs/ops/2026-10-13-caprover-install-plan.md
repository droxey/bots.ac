# CapRover fresh install on manage.bots.ac: implementation plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Harden manage.bots.ac (<server-ip>), then install CapRover v1.15.4 fresh, activate Pro, and deploy one test app from GitHub Actions through GHCR, without touching Postgres, audiovault-api, Tailscale or SSH.

**Architecture:** Phase 0 locks the host down: key-only SSH, fail2ban and a default-deny ufw. Docker CE (single-node swarm) then runs CapRover at `dev.bots.ac`, with host nginx and certbot disabled so CapRover's nginx owns 80/443. CI builds images in GitHub Actions, pushes them to GHCR, and deploys them by digest with a per-app token.

**Tech Stack:** Ubuntu 24.04.5, ufw, fail2ban, Docker CE 25+, CapRover v1.15.4, caprover CLI 2.4.4, caprover/deploy-from-github v2.0.0, GHCR, Cloudflare DNS, 1Password CLI (`op`), age.

## Global constraints

- Host: `ssh manage.bots.ac` (<server-ip>, user dani, port <ssh-port>). Server hostname `api.audiovau.lt`. Tailscale name `<server-tailnet-name>` (<server-tailnet-ip>).
- Out of scope, configs left as-is per Dani (ports <pg-port> and <api-port> closed publicly by ufw in Task 0.5): `postgresql@16-main`, `audiovault-api.service`, `audiovault-health-check`, and everything audiovau.lt. This plan does not change their config; they must still be `active` after each gated step.
- Keep running: `tailscaled`, `ssh` on <ssh-port>.
- Never run `apt remove`/`purge` on postgresql*, tailscale or openssh-server. Never touch `/var/lib/postgresql` or `/home/dani/.ssh/authorized_keys`.
- Every firewall change keeps <ssh-port>/tcp and the `tailscale0` interface open, and is followed by a fresh SSH login test. Keep one SSH session open during every SSH or firewall step. Tailscale SSH (`ssh dani@<server-tailnet-name>`) is the fallback.
- Admin password (18 chars) lives only in 1Password: `op://<vault>/<item-id-redacted>/password` (account musexmachine, vault Nebula, item "CapRover admin - manage.bots.ac", website `https://captain.dev.bots.ac`). Read it on the box with the token from `<op-token-file>`, inside a subshell, without `export`. Never echo it, never pass it as an argument, never write it to a file that outlives the step.
- `captain42` must be rotated in the same gated step as the install (Task 4). The dashboard port <setup-port> is reachable only from the box's egress IP until then.
- DNS changes in Cloudflare zone `bots.ac` are made by SysOps (the Cloudflare owner), not by the executor.
- Pin by digest or commit SHA: `caprover/caprover:1.15.4@sha256:6f3cb84688b0c43d2bfe06b212d280cce5b0a1f78f30c73e8505aeca0d05eafe`, `actions/checkout@11d5960a326750d5838078e36cf38b85af677262` (v4), `docker/login-action@c94ce9fb468520275223c153574b00df6fe4bcc9` (v3), `docker/build-push-action@10e90e3645eae34f1e60eeb005ba3a3d33f178e8` (v6), `caprover/deploy-from-github@058a3e0d8fb21bfc0ce207b79b1fc757d765a009` (v2.0.0). All resolved 2026-10-13.
- Versions verified 2026-10-13: CapRover v1.15.4 (https://github.com/caprover/caprover/releases), CLI `caprover@2.4.4` (https://registry.npmjs.org/caprover/latest), deploy-from-github v2.0.0 with inputs `server`, `token`, `app`, `image` (https://github.com/caprover/deploy-from-github/blob/v2.0.0/action.yml).
- Docs: https://caprover.com/docs/get-started.html, https://caprover.com/docs/troubleshooting.html, https://caprover.com/docs/firewall.html, https://docs.docker.com/engine/network/packet-filtering-firewalls/.
- Backups are encrypted with age before they land on the box (key in 1Password item "manage.bots.ac backup age key").
- Every step marked **STOP-AND-CONFIRM** needs Dani's explicit go-ahead in chat before it runs.

## Decisions (Dani, 2026-10-13)

- Root domain `dev.bots.ac`. Wildcard `*.dev.bots.ac A <server-ip>`, DNS only. Dashboard: `https://captain.dev.bots.ac`.
- api.audiovau.lt is out of scope. Host nginx and its certbot timer are disabled. No proxy app.
- No existing apps to migrate. PR previews: yes, as the last phase.
- Certificate email `dani@musexmachine.com`. GHCR user `droxey`.

## Baseline (2026-10-13, read-only)

Ubuntu 24.04.5, kernel 6.8.0-139 (6.8.0-146 installed, reboot pending), 29 pending updates, 16 vCPU, 15 GiB RAM, no swap, 451G free. No Docker, no `/captain`. Public listeners: nginx 80/443, sshd <ssh-port> (password auth effectively on via `50-cloud-init.conf`), plus Postgres <pg-port> and audiovault <api-port> (out of scope). ufw inactive, no fail2ban, journal 2.3 GB, `~/linux-doctor-tmp` left over from the health audit. Sources: `/workspace/reports/2026-10-13-manage-bots-ac-caprover-inventory.txt`, `/workspace/reports/2026-10-13-manage-bots-ac-health.md`, `/workspace/reports/2026-10-13-caprover-plan-security-audit.md`.

---

## Phase 0: Host hardening (before CapRover)

### Task 0.1: Encrypted backup of everything Phase 0 touches (DONE 2026-10-13, 15:40 ET: age key stored in 1Password item `op://<vault>/<item-id-redacted>` ("manage.bots.ac backup age key"), local copy shredded, public key at `/workspace/backups/age.pub`, `/etc` backup `etc.tgz.age` (44 KB) decrypt-tested; optional pg dump skipped)

- [ ] **Step 1: Create the age key once (box)**

```bash
mkdir -p -m 700 /home/box/.secrets/age /workspace/backups/manage-bots-ac-2026-10-13
chmod 700 /workspace/backups/manage-bots-ac-2026-10-13
( umask 077; age-keygen -o /home/box/.secrets/age/manage-bots-ac.key 2>/workspace/backups/age.pub.tmp )
sed -n 's/^Public key: //p' /workspace/backups/age.pub.tmp > /workspace/backups/age.pub && rm /workspace/backups/age.pub.tmp
```

Install `age` on the box first if missing (`sudo apt-get install -y age`). Store the private key in 1Password as "manage.bots.ac backup age key", then delete `/home/box/.secrets/age/manage-bots-ac.key`. Restores fetch it with `op read`.
Verify: `grep -c '^age1' /workspace/backups/age.pub` returns `1`.

- [ ] **Step 2: Pull /etc config, encrypted**

```bash
B=/workspace/backups/manage-bots-ac-2026-10-13; R=$(cat /workspace/backups/age.pub)
ssh manage.bots.ac 'sudo tar czf - /etc/ssh /etc/nginx /etc/letsencrypt /etc/systemd/journald.conf /etc/ufw' | age -r "$R" > $B/etc.tgz.age
```

Verify: `ls -l $B/etc.tgz.age` is non-empty. Decrypt test with the key from 1Password: `op read "op://<vault>/<item-id-redacted>/password" | age -d -i - $B/etc.tgz.age | tar tz | grep -c etc/ssh/sshd_config` returns a non-zero count.
Optional: an encrypted `pg_dumpall` as an extra safety copy (`ssh manage.bots.ac 'sudo -u postgres pg_dumpall' | gzip | age -r "$R" > $B/pg_dumpall.sql.gz.age`). It isn't required, because no step changes Postgres.
Rollback: none needed (read-only).

### Task 0.2: Firewall backend check (read-only) (DONE 2026-10-13, 15:41 ET: `iptables-nft`, 7 ts-input rules in nft, 0 in legacy; backends consistent)

- [ ] **Step 1: Confirm Tailscale uses the nft backend that Docker will use**

Run: `ssh manage.bots.ac 'update-alternatives --query iptables | grep ^Value; sudo iptables-nft -S | grep -c ts-input; sudo iptables-legacy -S 2>/dev/null | grep -c ts-input'`
Expected: `Value: /usr/sbin/iptables-nft`, a non-zero count, then `0`. If Tailscale's chains appear only in legacy, stop and report: Docker, ufw and Tailscale must share one backend before any later task runs.
Rollback: none (read-only).

### Task 0.3: Key-only SSH (DONE 2026-10-13, 15:48 ET, approved by Dani 15:47 ET: effective settings passwordauthentication no, kbdinteractiveauthentication no, maxauthtries 3, x11forwarding no; fresh key login OK; password/keyboard-interactive login refused with `Permission denied (publickey)`; backup at `/root/50-cloud-init.conf.bak`)

- [ ] **Step 1: STOP-AND-CONFIRM, then fix the cloud-init override** (a mistake can lock out SSH)

Keep the current session open.

```bash
ssh manage.bots.ac 'sudo cp /etc/ssh/sshd_config.d/50-cloud-init.conf /root/50-cloud-init.conf.bak && printf "PasswordAuthentication no\nKbdInteractiveAuthentication no\n" | sudo tee /etc/ssh/sshd_config.d/50-cloud-init.conf && printf "MaxAuthTries 3\nX11Forwarding no\n" | sudo tee /etc/ssh/sshd_config.d/10-hardening.conf && sudo sshd -t && sudo systemctl reload ssh'
```

- [ ] **Step 2: Verify from a second session before closing the first**

Run: `ssh manage.bots.ac 'sudo sshd -T | grep -E "^(passwordauthentication|kbdinteractiveauthentication|maxauthtries|x11forwarding) "'` then `ssh -o PreferredAuthentications=password -o PubkeyAuthentication=no -o ConnectTimeout=15 manage.bots.ac true`
Expected: `no`, `no`, `3`, `no`. The second command fails with `Permission denied`. A fresh key login still works.
Rollback: `sudo cp /root/50-cloud-init.conf.bak /etc/ssh/sshd_config.d/50-cloud-init.conf && sudo rm /etc/ssh/sshd_config.d/10-hardening.conf && sudo systemctl reload ssh`, run from the open session or over Tailscale SSH.

### Task 0.4: fail2ban for <ssh-port> (DONE 2026-10-13, 15:41 ET: fail2ban active on <ssh-port> (systemd backend), ignoreip 127.0.0.0/8, ::1, 100.64.0.0/10 and box egress <box-egress-ip>; 1 IP banned on first scan. The box egress IP may change; update `ignoreip` if it does)

- [ ] **Step 1: Install fail2ban and add the sshd jail**

```bash
ssh manage.bots.ac 'sudo apt-get install -y fail2ban && printf "[sshd]\nenabled = true\nport = <ssh-port>\nbackend = systemd\nmaxretry = 5\nbantime = 1h\nignoreip = 127.0.0.1/8 ::1 100.64.0.0/10\n" | sudo tee /etc/fail2ban/jail.d/sshd.local && sudo systemctl enable --now fail2ban'
```

Look up the box's egress IP at run time (`curl -s https://ifconfig.me`) and add it to `ignoreip`, so the executor can't ban itself.
Verify: `ssh manage.bots.ac 'sudo fail2ban-client status sshd'` lists the jail, and a fresh SSH login works.
Rollback: `sudo systemctl disable --now fail2ban && sudo apt-get purge -y fail2ban`.

### Task 0.5: Default-deny ufw (DONE 2026-10-13 15:25 ET, approved by Dani 14:52 ET)

- [x] **Step 1: Back up firewall state.** `iptables-save`, `ip6tables-save`, `nft list ruleset`, `ufw status verbose` and `/etc/ufw` saved to `~/fw-backup-2026-10-13/` on the server and copied to `/workspace/reports/fw-backup-2026-10-13/` on the box.
- [x] **Step 2: Enable ufw**, with a second SSH session open and a 5-minute `systemd-run` timer that would run `ufw disable` (cancelled after verification):

```bash
sudo ufw default deny incoming && sudo ufw default allow outgoing && sudo ufw allow <ssh-port>/tcp && sudo ufw allow in on tailscale0 && sudo ufw allow <tailscale-udp-port>/udp && sudo ufw allow 80/tcp && sudo ufw allow 443/tcp && sudo ufw allow 443/udp && sudo ufw deny <pg-port>/tcp && sudo ufw deny <api-port>/tcp && sudo ufw deny <swarm-mgmt-port>/tcp && sudo ufw deny <swarm-gossip-port>/tcp && sudo ufw deny <swarm-gossip-port>/udp && sudo ufw deny <vxlan-port>/udp && sudo ufw --force enable
```

<pg-port> and <api-port> are closed publicly on Dani's explicit approval, even though audiovau.lt may break. Postgres and audiovault configs and services were not touched. Tailscale clients can still reach them over `tailscale0`.
- [x] **Step 3: Verified from the box.** A new SSH session on <ssh-port> works. 80 and 443 accept connections. <pg-port>, <api-port>, <swarm-mgmt-port> and <swarm-gossip-port> time out (filtered). UDP <swarm-gossip-port>/<vxlan-port> can't be probed from the box. `tailscale status` is up, and `ssh`, `postgresql@16-main`, `audiovault-api`, `tailscaled` and `ufw` are all `active`.

Rollback: `sudo ufw disable`. To restore the exact earlier state, also restore `/etc/ufw` from `fw-backup-2026-10-13/etc-ufw.tgz`.

### Task 0.6: Cap journald and clean up (DONE 2026-10-13, 15:41 ET: journald capped at 500M (494.1M after vacuum), `~/linux-doctor-tmp` removed)

- [ ] **Step 1: Cap journald at 500M**

Run: `ssh manage.bots.ac 'sudo mkdir -p /etc/systemd/journald.conf.d && printf "[Journal]\nSystemMaxUse=500M\n" | sudo tee /etc/systemd/journald.conf.d/10-size.conf && sudo systemctl restart systemd-journald && sudo journalctl --vacuum-size=500M'`
Verify: `ssh manage.bots.ac 'journalctl --disk-usage'` reports 500M or less.
Rollback: `sudo rm /etc/systemd/journald.conf.d/10-size.conf && sudo systemctl restart systemd-journald` (vacuumed logs are gone for good).

- [ ] **Step 2: Remove the Linux Doctor leftovers**

Run: `ssh manage.bots.ac 'rm -rf ~/linux-doctor-tmp /tmp/ld.tgz; ls -d ~/linux-doctor-tmp'`
Expected: `No such file or directory`.
Rollback: not needed (the tool can be re-fetched from https://github.com/zShaD0w7x/linux-doctor).

### Task 0.7: Updates and reboot (DONE 2026-10-13, approved by Dani 15:47 ET: `apt-get -y upgrade` with confold, 58 package actions, RC=0; rebooted 15:49 ET, SSH back 15:51 ET; kernel 6.8.0-146-generic; all six services active; 24 ufw rules intact; sshd settings unchanged. Left over: 4 kernel meta-packages to 6.8.0-147 held back by `upgrade` (need `apt-get full-upgrade` plus another reboot))

- [ ] **Step 1: STOP-AND-CONFIRM, then apply the 29 updates and reboot** (several minutes of downtime for everything on the host)

Run: `ssh manage.bots.ac 'sudo apt-get update && sudo apt-get -y upgrade && sudo reboot'`
Verify (after about 2 min): `ssh -o ConnectTimeout=15 manage.bots.ac 'uname -r; systemctl is-active postgresql@16-main audiovault-api tailscaled ssh fail2ban ufw; sudo ufw status | head -3; apt list --upgradable 2>/dev/null | wc -l'`
Expected: `6.8.0-146-generic` or newer, `active` six times, ufw active, and `1` (the header line only). This also confirms every Phase 0 change survives a reboot.
Rollback: pick the previous kernel (6.8.0-139) from the GRUB menu in the Servarica console, then `sudo apt-mark hold linux-image-generic`.

---

## Phase 1: CapRover

### Task 1: Preflight and DNS

- [ ] **Step 1: Confirm nothing to uninstall**

Run: `ssh -o ConnectTimeout=15 manage.bots.ac 'command -v docker; ls -d /captain; systemctl is-active postgresql@16-main audiovault-api tailscaled'`
Expected: no docker path, `ls: cannot access '/captain'`, then `active` three times. If docker or /captain exists, run Task 2. Otherwise mark Task 2 as a no-op.

- [ ] **Step 2: DNS from SysOps**

Ask SysOps (Cloudflare owner) to create `*.dev.bots.ac` A <server-ip>, proxy status DNS only (staging / CapRover root). For production: keep `*.bots.ac` A <server-ip>, and also add per-app records plus an apex record for `bots.ac`. Guard: CapRover's catch-all must return 404 for unknown hostnames. Decision (Dani 2026-10-16).
Verify: `curl -s -H 'accept: application/dns-json' 'https://cloudflare-dns.com/dns-query?name=captain.dev.bots.ac&type=A'` contains `"data":"<server-ip>"`. The box resolver returns <box-resolver-ip>, so always check through DoH. Also confirm a random unused name under `*.bots.ac` resolves but returns 404 from CapRover once nginx is up.
Rollback: SysOps deletes the record.

### Task 2: Uninstall an existing CapRover (only if Task 1 found one)

- [ ] **Step 1: Record and back up, encrypted**

Run: `B=/workspace/backups/manage-bots-ac-2026-10-13; ssh manage.bots.ac 'sudo docker service ls; sudo docker volume ls; sudo du -sh /captain' > $B/caprover-before.txt && ssh manage.bots.ac 'sudo tar czf - /captain' | age -r "$(cat /workspace/backups/age.pub)" > $B/captain.tgz.age`
Verify: `ls -l $B/captain.tgz.age` is non-empty.

- [ ] **Step 2: STOP-AND-CONFIRM, then run the official uninstall** (destructive)

```bash
ssh manage.bots.ac 'sudo docker service rm $(sudo docker service ls -q); sudo rm -rf /captain; sudo docker swarm leave --force; sudo docker system prune --all --force'
```

Verify: `ssh manage.bots.ac 'sudo docker service ls -q | wc -l; ls -d /captain; systemctl is-active postgresql@16-main audiovault-api tailscaled'` gives `0`, `No such file or directory`, then `active` three times.
Rollback: decrypt `captain.tgz.age` onto `/`, then re-run Task 4 Step 2.

### Task 3: Docker CE and Docker-aware firewall

- [ ] **Step 1: STOP-AND-CONFIRM, then install Docker CE with a fingerprint check** (changes iptables)

```bash
ssh manage.bots.ac 'sudo install -m 0755 -d /etc/apt/keyrings && sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc && gpg --show-keys /etc/apt/keyrings/docker.asc | tr -d " " | grep -q 9DC858229FC7DD38854AE2D88D81803C0EBFCD88 && echo "deb [arch=amd64 signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu noble stable" | sudo tee /etc/apt/sources.list.d/docker.list && sudo apt-get update && sudo apt-get install -y docker-ce docker-ce-cli containerd.io && sudo apt-mark hold docker-ce docker-ce-cli containerd.io'
```

Verify: `ssh manage.bots.ac 'sudo docker version --format {{.Server.Version}}; tailscale status | head -2; systemctl is-active postgresql@16-main audiovault-api; sudo iptables-legacy -S 2>/dev/null | grep -c DOCKER'`
Expected: 25 or higher, Tailscale peers, `active` twice, and `0` (Docker uses nft like Tailscale).
Rollback: `sudo apt-mark unhold docker-ce docker-ce-cli containerd.io && sudo apt-get purge -y docker-ce docker-ce-cli containerd.io && sudo rm -rf /var/lib/docker /etc/apt/sources.list.d/docker.list`, then reboot to clear Docker's chains.

- [ ] **Step 2: Persist the DOCKER-USER rule for port <setup-port>**

Docker-published ports skip ufw, so this rule filters them. The unit only touches DOCKER-USER; ufw keeps owning INPUT (<ssh-port>, tailscale0, swarm denies).

```bash
BOXIP=$(curl -s https://ifconfig.me)
ssh manage.bots.ac "printf '[Unit]\nAfter=docker.service\nRequires=docker.service\n[Service]\nType=oneshot\nRemainAfterExit=yes\nExecStart=/usr/sbin/iptables -I DOCKER-USER -p tcp -m conntrack --ctorigdstport <setup-port> --ctdir ORIGINAL ! -s $BOXIP -j DROP\nExecStop=/usr/sbin/iptables -D DOCKER-USER -p tcp -m conntrack --ctorigdstport <setup-port> --ctdir ORIGINAL ! -s $BOXIP -j DROP\n[Install]\nWantedBy=multi-user.target\n' | sudo tee /etc/systemd/system/fw-caprover.service && sudo systemctl daemon-reload && sudo systemctl enable --now fw-caprover"
```

Verify: `ssh manage.bots.ac 'sudo iptables -S DOCKER-USER'` shows the DROP rule, and a fresh SSH login works.
Rollback: `sudo systemctl disable --now fw-caprover && sudo rm /etc/systemd/system/fw-caprover.service`.

- [ ] **Step 3: Free ports 80/443 (STOP-AND-CONFIRM; api.audiovau.lt goes offline and is not restored)**

Run: `ssh manage.bots.ac 'sudo systemctl disable --now nginx certbot.timer && sudo ss -tlnp | grep -cE ":(80|443) "; systemctl is-active postgresql@16-main audiovault-api tailscaled ssh'`
Expected: `0`, then `active` four times.
Rollback: `sudo systemctl enable --now nginx certbot.timer`.

### Task 4: Install CapRover and run setup in one gated step

- [ ] **Step 1: Install the CLI on the box**

Run: `npm install -g caprover@2.4.4 && caprover --version`
Expected: `2.4.4`.

- [ ] **Step 2: STOP-AND-CONFIRM, then install and immediately run serversetup** (no pause between the two commands; `captain42` is live only in between, and only from the box IP)

```bash
ssh manage.bots.ac 'sudo docker run -p 80:80 -p 443:443 -p <setup-port>:<setup-port> -e ACCEPTED_TERMS=true -v /var/run/docker.sock:/var/run/docker.sock -v /captain:/captain caprover/caprover:1.15.4@sha256:6f3cb84688b0c43d2bfe06b212d280cce5b0a1f78f30c73e8505aeca0d05eafe'
until curl -s -o /dev/null -m 5 -w '%{http_code}' http://<server-ip>:<setup-port> | grep -q 200; do sleep 5; done
(
  umask 077
  D="$(mktemp -d -p /home/box/agent-data)"; trap 'rm -rf "$D"' EXIT
  OP_SERVICE_ACCOUNT_TOKEN="$(cat <op-token-file>)" op read "op://<vault>/<item-id-redacted>/password" \
    | python3 -c 'import sys,json; json.dump({"caproverIP":"<server-ip>","caproverPassword":"captain42","caproverRootDomain":"dev.bots.ac","newPassword":sys.stdin.read().rstrip("\n"),"certificateEmail":"dani@musexmachine.com","caproverName":"manage-bots-ac"}, open(sys.argv[1],"w"))' "$D/setup.json"
  caprover serversetup -c "$D/setup.json"
)
caprover logout -n manage-bots-ac
```

The 1Password service account should be read-only on vault Nebula. Never run this with `set -x`. serversetup in CLI 2.4.4 forces HTTPS before it changes the password (`built/commands/serversetup.js`).
Verify: `ssh manage.bots.ac 'sudo docker service ls --format "{{.Name}} {{.Image}} {{.Replicas}}"'`, then `curl -sI https://captain.dev.bots.ac | head -1` and `curl -s -o /dev/null -w '%{http_code}\n' http://captain.dev.bots.ac`
Expected: `captain-captain` and `captain-nginx` at `1/1`, `HTTP/2 200`, then `301` or `302`. Logging in with `captain42` fails. `ls /home/box/agent-data/tmp.* 2>/dev/null` is empty, and the box has no CapRover login saved (`caprover list` shows none).
Rollback: Task 2 Step 2 commands, then `sudo systemctl enable --now nginx certbot.timer`.

- [ ] **Step 3: Close port <setup-port> to everyone**

Run: `ssh manage.bots.ac 'sudo sed -i "s/ ! -s [0-9.]*//" /etc/systemd/system/fw-caprover.service && sudo systemctl daemon-reload && sudo systemctl restart fw-caprover && sudo iptables -S DOCKER-USER'`, then from the box `curl -m 5 -s -o /dev/null -w '%{http_code}\n' http://<server-ip>:<setup-port>`
Expected: the rule has no source exception and the curl prints `000`. The DOCKER-USER rule is the control that matters; recheck it after every CapRover update. A new SSH login and `tailscale status` still work.
Rollback: `sudo systemctl stop fw-caprover` (reopens <setup-port>).

### Task 5: Pro, login alerts, 2FA (right after Task 4, nothing in between)

- [ ] **Step 1: STOP-AND-CONFIRM, then enter the Pro key** (paid license, $23, bought 2026-10-13)

In `https://captain.dev.bots.ac` go to Settings → CapRover Pro and paste the key from Dani's purchase email.
Verify: the Pro section shows active.

- [ ] **Step 2: Turn on login and build-failure alerts** (Pro → Notifications, email to dani@musexmachine.com)

Verify: a fresh browser login triggers a login alert email.

- [ ] **Step 3: Enable 2FA and store the TOTP seed in the 1Password item**

Verify: after logging out and back in, an OTP prompt appears.

- [ ] **Step 4: Limit the dashboard to the box, the tailnet, and GitHub Actions (recommended)**

In Settings → NGINX config, allow `<box egress IP>`, `100.64.0.0/10`, and the current GitHub Actions IP ranges from `https://api.github.com/meta` (`actions` key), then `deny all;` on the `captain.dev.bots.ac` server block. Auto-refresh the Actions ranges on a schedule (for example a weekly cron that rewrites the allow lines from `meta` and reloads nginx). Decision (Dani 2026-10-16). Residual risk: those ranges are shared by all GitHub users, so protection rests on the CapRover password, 2FA and a deploy-only app token.
Verify: from the box, `curl -sI https://captain.dev.bots.ac | head -1` returns `HTTP/2 200`; from a non-GitHub, non-tailnet network the dashboard returns 403; a GitHub Actions deploy reaches the dashboard.
Rollback: restore the default NGINX config in the same screen. If that locks the dashboard out, follow "I've made a change to the Nginx config that broke the admin UI!" at https://caprover.com/docs/troubleshooting.html. Disable 2FA and alerts in the Pro screen; the TOTP seed stays in 1Password.

### Task 6: GHCR registry and test app via GitHub Actions

- [ ] **Step 1: Add GHCR as a private registry with a scoped reader token**

Create a classic PAT on a dedicated bot account that can see only the needed packages (GHCR does not take fine-grained PATs). Give it only `read:packages` and a 90-day expiry, store it in 1Password with a rotation reminder, and never give it `write:packages` or `repo`. In Cluster → Docker Registry Configuration, add `ghcr.io` with that account's username and the PAT. If a bot account is not set up yet, use user `droxey` with the same scope and expiry.
Verify: the registry is listed with no error. Never enable the self-hosted registry (port <registry-port>).

- [ ] **Step 2: Create app `hello`, enable its app token, scope it to an Environment**

Dashboard: Apps → create `hello`, Deployment → enable App Token. In the GitHub repo, create Environment `production` limited to branch `main`, and add the Environment secrets `CAPROVER_APP_TOKEN` (the token) and `CAPROVER_SERVER` (`https://captain.dev.bots.ac`).
Verify: Settings → Environments shows `production` with the branch rule and both secrets.

- [ ] **Step 3: Add `.github/workflows/deploy.yml`, SHA-pinned, deploying by digest**

```yaml
name: deploy
on: { push: { branches: [main] } }
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production
    permissions: { contents: read, packages: write }
    steps:
      - uses: actions/checkout@11d5960a326750d5838078e36cf38b85af677262 # v4
      - uses: docker/login-action@c94ce9fb468520275223c153574b00df6fe4bcc9 # v3
        with: { registry: ghcr.io, username: ${{ github.actor }}, password: ${{ secrets.GITHUB_TOKEN }} }
      - id: build
        uses: docker/build-push-action@10e90e3645eae34f1e60eeb005ba3a3d33f178e8 # v6
        with: { push: true, tags: "ghcr.io/${{ github.repository }}:${{ github.sha }}" }
      - uses: caprover/deploy-from-github@058a3e0d8fb21bfc0ce207b79b1fc757d765a009 # v2.0.0
        with:
          server: ${{ secrets.CAPROVER_SERVER }}
          app: hello
          token: ${{ secrets.CAPROVER_APP_TOKEN }}
          image: ghcr.io/${{ github.repository }}@${{ steps.build.outputs.digest }}
```

Verify: the Actions run succeeds. After enabling HTTPS for `hello`, `curl -s -o /dev/null -w '%{http_code}\n' https://hello.dev.bots.ac` returns `200`.
Rollback: delete the workflow and the app, then revoke the app token and the GHCR PAT.

### Task 7: Encrypted backups

Decision (Dani 2026-10-16): primary design is the operating-rules restic path (Mac pull to iCloud plus RackNerd), with the bots.ac DB and attachments included. Drop R2. The Grok Bot box copy at `~/backups/grokbot` on the server is always kept, never dropped.

- [ ] **Step 1: First CapRover backup (into the always-kept box copy)**

Take a backup in Settings → Backup, download it, then store an age-encrypted copy under the server's `~/backups/grokbot/` tree (and optionally mirror to `/workspace/backups/manage-bots-ac-2026-10-13/` on the box). Example: `age -r "$(cat /workspace/backups/age.pub)" -o ~/backups/grokbot/caprover-backup.tar.age <downloaded file> && shred -u <downloaded file>` (run the write on the server path via SSH as needed).
Verify: the `.age` file under `~/backups/grokbot/` is non-empty, and a decrypt test with the 1Password key lists `config-captain.json`. Confirm `~/backups/grokbot` still exists and was not pruned.

- [ ] **Step 2: Align with restic (rules section 6); keep the box copy**

Wire CapRover `/captain/data` and the bots.ac DB dumps into the restic/Backrest job from the operating rules. Do not add R2. Keep publishing a weekly encrypted snapshot into `~/backups/grokbot` on the server (age tarball of `/captain/data` is fine); that path is never dropped when pruning other backup targets.
Verify: restic includes the bots.ac DB dump; `ls ~/backups/grokbot` on the server still shows recent files after prune; no R2 remote is configured.
Rollback: remove any new cron/restic paths that are wrong; never delete `~/backups/grokbot`.

### Task 8 (last phase): PR previews

Entry criterion: a test shows `caprover` CLI 2.4.4 can create and delete a throwaway app non-interactively with 2FA on (`CAPROVER_OTP_TOKEN`). Until it passes, Task 8 does not start.

- [ ] **Step 1: Preview workflow rules**
  - Trigger `pull_request` only, with `if: github.event.pull_request.head.repo.full_name == github.repository`. Never use `pull_request_target`, and never run fork code with secrets.
  - Keep the admin credential in Environment `previews` with required reviewers.
  - App name `pr-<number>-<repo>`, served at `pr-<number>-<repo>.dev.bots.ac`. No secrets or customer names in branch-derived hostnames, because certificates are public in CT logs.
  - Per-app limits: memory 512M and 0.5 CPU (`docker service update --limit-memory 512M --limit-cpu 0.5 srv-captain--pr-<n>-<repo>`).
  - At most 3 previews at once. Let's Encrypt allows 50 certificates per registered domain per 7 days (https://letsencrypt.org/docs/rate-limits/).
- [ ] **Step 2: Cleanup.** Delete the app on `pull_request: closed`, plus a daily scheduled sweep that deletes `pr-*` apps older than 7 days.

Verify: open a same-repo test PR and confirm the preview responds over HTTPS. Close it and confirm the app is gone. Then open a PR from a fork and confirm no preview job runs.
Rollback: disable the workflow and delete every `pr-*` app.

## Decommission and full rollback checklist

SysOps deletes `*.dev.bots.ac` (and any production `*.bots.ac` / per-app / apex records that pointed at this host), so a released IP can't be taken over. Then run Task 2 Step 2, `sudo systemctl disable --now fw-caprover`, revoke the GHCR PAT and app tokens, disable 2FA and remove the Pro key from the old install, and reboot to clear Docker's iptables chains. Restoring Phase 0 means running each task's own rollback in reverse order.

## Risks not verified

- Whether `caprover serversetup -c` accepts every key above in 2.4.4. The key names come from the CLI source, but the command has not been run.
- Whether CapRover re-adds port <setup-port> to `captain-captain` on update. The DOCKER-USER rule covers it either way.
- CLI with 2FA (`CAPROVER_OTP_TOKEN`) for previews.

## Gaps / not covered

- Monitoring and alerting: no uptime checks for `captain.dev.bots.ac`, CapRover apps or the host, and no disk or memory alerts.
- Offsite encrypted backups: Task 7 follows the restic design (Mac + iCloud + RackNerd) and always keeps `~/backups/grokbot` on the server; R2 is dropped. A scheduled restore test for `/captain` is still needed.
- CapRover update policy: no cadence, changelog review or digest bump procedure for CapRover, its nginx and certbot images, or the held Docker packages.
- Ansvisor deploy: queued after this install. Not planned here.
- Secret cleanup on the box: the 1Password service-account token file, the CLI configstore and any leftover temp files have no audit or rotation schedule. The service account's vault scope is not yet confirmed as read-only on Nebula.
- Postgres (<pg-port>) and audiovault (<api-port>) are now closed publicly (Task 0.5). Any public client of audiovau.lt's database or API on this host has lost access.
- Passwordless sudo for the `sudo` group and root's unused authorized key (health audit items 5 and 7).
- Swap file for Docker builds, and pruning the offline Tailscale peer `<offline-peer>`.
