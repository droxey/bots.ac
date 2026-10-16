# bots.ac Operating and Security Rules (PLAN ONLY)
Date: Fri Oct 16, 2026 · Server: manage.bots.ac (Servarica, Montreal) · Ubuntu 24.04 · 16 vCPU · 15 GiB RAM
Status: **draft.** Nothing has been installed, changed, or SSH'd. Every step below waits for Dani's go.

**Goal:** Run CapRover Pro, which stays our PaaS (no Coolify, no Dokploy), plus a small set of apps on one box. Only websites are public. Everything else stays private. A mistake should cost minutes, not data.

**Skills used:** `superpowers/brainstorming` (I looked at the context first, compared 2–3 approaches, and wrote questions for Dani), `superpowers/writing-plans` (ordered, testable steps, each with a check and a rollback), and `workflows/sshid-authorized-keys` (safe key install for rule 13). The workflows folder had no skill named security, hardening, docker, caprover or premortem. I wrote the premortem in section 11 by hand.

Words used here:
- **Tailnet IP** is the box's private Tailscale address (100.x.y.z).
- **Public** means anyone on the internet can reach it.
- **Private** means only reachable over Tailscale, an SSH tunnel, or 127.0.0.1.

---

## Decisions locked (2026-10-16)
Dani said "Yes, use all recommended" at 4:53 AM ET. Final choices:

Premortem decisions 1-6 (same day; see `2026-10-16-premortem.md`): SSH stays on the current `<ssh-port>` (rules previously used a wrong one-digit-different value); keep `*.bots.ac` plus per-app and apex records with CapRover catch-all 404; dashboard `captain.dev.bots.ac`; CI via auto-refreshed GitHub Actions IP ranges from `api.github.com/meta`; Postgres on localhost plus the tailnet IP after a fresh `pg_dump`; restic design plus the bots.ac DB, drop R2, always keep `~/backups/grokbot` on the server. Questions 7-20 remain open.

- **Q1:** Close Postgres <pg-port> first, before anything else, after a fresh dump.
- **Q2:** Bots use a `bot` user, own key, sudo only for read-only commands.
- **Q3:** Public key-only SSH on <ssh-port> + Servarica console + console password in 1Password.
- **Q4:** fail2ban (simple, local).
- **Q5:** Free Healthchecks.io/UptimeRobot plus heartbeat.
- **Q6:** Already answered — Mac + iCloud (see Q-B).
- **Q7:** Keep IPv6; mirror every rule in ip6tables and test.
- **Q8:** Samba serves Time Machine with a 200G cap (already decided).
- **Q9:** ntfy public endpoint with auth tokens; admin private.
- **Q10:** Only sites, Kuma status page, and error ingest public; all admin UIs tailnet-only.
- **Q11:** UTC on the server; ET in reports.
- **Q12:** Unattended reboots yes — security-only, 5:00 AM ET.
- **Q13:** Yes, back up Syncthing data on the server.
- **Q14:** Replaced by Q-D (answered below).
- **Q-A:** Time Machine cap **200G**.
- **Q-B:** Mac **pulls** the restic repo nightly (rrsync + launchd + pmset + Healthchecks).
- **Q-C:** Yes — second copy on spare RackNerd via restic over SFTP weekly.
- **Q-D:** Dani already has the **iCloud 2 TB** plan; no upgrade needed.
- **Q-E:** MacBook sleeps plugged in (lid closed or open); wake at 04:55 works.

---

## 0. Ground rules for bots, including me
1. **Bots plan, Dani approves.** A bot never runs anything on bots.ac that changes state (installs, firewall, sshd, docker, deletes, DB writes, reboots) unless Dani has approved that exact command list in chat.
2. **No destructive commands without a backup and a second "yes".** This covers `rm -rf`, `docker system prune -a`, `docker volume rm`, `DROP`/`TRUNCATE`, `ufw reset`, `iptables -F`, `mkfs`, `dd`, editing `sshd_config`, removing a CapRover app, and `apt purge`. Before any of these, the bot (a) shows the exact command, (b) confirms a backup less than 24 hours old exists and was checked, and (c) waits for Dani to type "yes, run it".
3. **Bots log in with their own user and key, not Dani's.** Give them a `bot` user with only the sudo commands they need (read-only commands like `docker ps`, `journalctl`, `ufw status`, `df`). Bots get no sudo for `rm`, `iptables` or `docker rm`. Every bot session gets logged (`auditd` or `sudo` logs).
4. **Every change goes in a short change log** at `/root/CHANGELOG.md` and in a private git repo with no secrets in it. Each entry has the date, who made it, what changed, and how to undo it.
5. **Run heavy jobs with `nice`/`ionice`** (backups, scans, image builds) so the live apps stay fast.

## 1. Firewall (UFW + DOCKER-USER)
6. **Default policy: deny incoming, allow outgoing, deny routed.** Allow in from the public internet only on **<ssh-port>/tcp (SSH)**, **80/tcp**, **443/tcp**, and **<tailscale-udp-port>/udp (Tailscale direct connections, optional)**. Everything else gets blocked.
7. **Allow all traffic on the `tailscale0` interface** (`ufw allow in on tailscale0`). Private apps get reached this way.
8. **Docker skips UFW**, so we also filter in the **DOCKER-USER** iptables chain using the `ufw-docker` method (rules go in `/etc/ufw/after.rules` and `after6.rules`, so they survive reboots and UFW reloads). Those rules: allow established connections, allow from `tailscale0` and the docker/swarm internal networks, allow public 80/443 to the CapRover nginx, and **drop everything else from the public interface**, including <setup-port>, <registry-port>, <swarm-gossip-port>, <vxlan-port>, <swarm-mgmt-port>, <pg-port>, <syncthing-gui-port>, <syncthing-port> and <cockpit-port>.
9. **Rule of thumb for apps:** no container publishes a host port, apart from CapRover's own nginx (80/443). Apps get reached through CapRover's proxy. If an app truly needs a host port (Syncthing's <syncthing-port>), bind it to the **tailnet IP or 127.0.0.1**, never `0.0.0.0`.
10. **IPv6 gets the same rules.** Keep `IPV6=yes` in `/etc/default/ufw` and copy every DOCKER-USER rule to `ip6tables`. Test from outside over IPv6 (`nmap -6`) as well as IPv4. If we don't need IPv6, the other option is to turn it off for Docker. See Q7.
11. **Prove it from outside.** After every firewall change, run `nmap -Pn -p- manage.bots.ac` (v4 and v6) from a different server (for example the unused RackNerd box). Only 80, 443 and <ssh-port> should show as open.

## 2. Private access (Tailscale)
12. **Private only. These are never public:**
- **Cockpit (<cockpit-port>):** native. Set `ListenStream=<tailnet-ip>:<cockpit-port>` in a `cockpit.socket` override.
- **Samba (<smb-ports>):** native. Set `interfaces = lo tailscale0` and `bind interfaces only = yes`. UFW blocks <smb-ports> publicly.
- **Syncthing GUI (<syncthing-gui-port>) and sync (<syncthing-port> tcp/udp):** see rule 30.
- **CapRover dashboard:** see rule 20.
- **Postgres (<pg-port>):** see rule 24.
- **Admin UIs behind CapRover with HTTPS + auth and a tailnet-only allowlist:** Backrest, Beszel, Homepage, GlitchTip admin, Readeck, Kuma admin, Gotify/ntfy admin.

**Public, with HTTPS:**
- Kuma **status page** only.
- **ntfy/Gotify message endpoint**, only if phones must get pushes off-tailnet (needs auth tokens).
- **GlitchTip ingest**, only if browser apps send errors to it.
- Real sites (audiovau.lt etc.).

**Default for anything new is private** until Dani says otherwise.

13. **Admin UIs behind CapRover get two locks:** CapRover HTTP Basic Auth (or the app's own login with 2FA if it has one) **and** an nginx `allow 100.64.0.0/10; deny all;` block in the app's custom nginx config. Then even a leaked password isn't enough from the public internet. **Exception for the CapRover dashboard only (`captain.dev.bots.ac`):** also allow GitHub Actions IP ranges from `api.github.com/meta` (auto-refreshed), so CI can deploy. Residual risk: those ranges are shared by all GitHub users, so protection rests on the CapRover password, 2FA and a deploy-only token. Decision (Dani 2026-10-16).
14. **Tailscale is a convenience, not the only way in.** SSH on <ssh-port> stays reachable from the public internet, key-only (see rule 16). If Tailscale goes down, Dani still has public SSH. If SSH breaks too, she has Servarica's web console. **Don't** turn on Tailscale SSH as the only path, and don't bind sshd only to the tailnet IP.
15. **Turn off Tailscale key expiry for this server** in the admin console, so it doesn't drop off the tailnet after 180 days. Turn on Tailscale ACLs so only Dani's devices (not every shared node) can reach <smb-port>, <cockpit-port> and <syncthing-gui-port>.

## 3. SSH hardening
16. **Keys only, port <ssh-port>.** Settings: `PasswordAuthentication no`, `KbdInteractiveAuthentication no`, `PermitRootLogin no`, `PubkeyAuthentication yes`, `AllowUsers dani bot`, `MaxAuthTries 3`. On Ubuntu 24.04, sshd uses **socket activation**. Set the port in `ssh.socket` (`ListenStream=<ssh-port>`) with a systemd override, *as well as* checking `sshd_config.d`. If only `sshd_config` gets changed, the port can quietly stay at 22.
17. **Add keys the safe way** (sshid-authorized-keys skill). Download to a temp file, accept only `ssh-ed25519` lines, check `ssh-keygen -lf` against fingerprints Dani has pinned, add each key only if it's missing, then set `.ssh` to 700 and the file to 600. Never use `curl >> authorized_keys`.
18. **Lockout-proof change routine:**
   (a) Keep **two SSH sessions open**.
   (b) Run `sshd -t` before reloading.
   (c) Run `systemctl reload ssh`, not restart.
   (d) Test a **third, fresh** login from a new terminal before closing anything.
   (e) For firewall changes, schedule an automatic undo first: `echo "ufw disable" | at now + 10 minutes`, or `iptables-restore` from a saved copy. Cancel it (`atrm`) only after the new login works.
   (f) Know where **Servarica's VNC/console** login is before starting, and keep a **console password** for `dani`. The console doesn't use SSH, so password login still works there.
19. **Use fail2ban** with the sshd jail on port <ssh-port>, backend `systemd`, `bantime 1h`, rising to repeat-offender bans. Add Dani's home IP and the tailnet range `100.64.0.0/10` to `ignoreip` so she can't ban herself. (CrowdSec is the alternative. See Q4.)

## 4. CapRover specifics
20. **Ports:**
- **80/443:** public (nginx).
- **<setup-port>:** first-setup dashboard only. Block it publicly in DOCKER-USER **right after** setup. After that, reach the dashboard only at `captain.dev.bots.ac` over HTTPS, with the nginx allowlist from rule 13 (tailnet plus auto-refreshed GitHub Actions IP ranges). Decision (Dani 2026-10-16): dashboard hostname is `captain.dev.bots.ac`.
- **<registry-port>** (registry), **<swarm-gossip-port> tcp/udp**, **<vxlan-port>/udp** and **<swarm-mgmt-port>/tcp** (swarm): **never public**. This is a single-node swarm and needs none of them from outside.
21. **Change the default password `captain42` on first login, before anything else.** Store the new one in 1Password. Turn on CapRover Pro's 2FA/OTP right away.
22. **Turn on HTTPS for the root domain and "Force HTTPS"** before deploying any app. Keep the wildcard DNS record `*.bots.ac → server IP`, and also add per-app records plus an apex record for `bots.ac`. Guard: CapRover's catch-all must return 404 for unknown hostnames. CapRover root stays `dev.bots.ac` (`*.dev.bots.ac`). Each app gets its own Let's Encrypt cert. Decision (Dani 2026-10-16).
23. **CapRover's Docker install vs firewall:** Docker installs its own iptables chains on start and can bypass or clash with UFW. Two rules here. Install Docker *first* (from Docker's apt repo, version per CapRover docs). Then put the DOCKER-USER rules (rule 8) in place **before** running the CapRover container. Then run `nmap` again (rule 11). After **every Docker upgrade or reboot**, check that `iptables -L DOCKER-USER` still has the drop rule. A Beszel/Kuma check or a small systemd timer should alert if it's missing.

## 5. Postgres (legacy audiovault)
24. **Postgres listens on 127.0.0.1 plus the tailnet IP only** (and the docker bridge if CapRover apps need it; then restrict access in `pg_hba.conf` to localhost and the tailnet range). UFW denies <pg-port> publicly. Take a fresh `pg_dump` before changing listen addresses or `pg_hba.conf`. **Check right now, in plan step 1, whether <pg-port> is open to the internet.** If it is, that's the first fix, before anything else. Decision (Dani 2026-10-16).
25. Leave the audiovault service and its Postgres alone (native, outside CapRover) until it has a tested backup. Moving them into CapRover is a separate, later plan.

## 6. Backups (Backrest/restic)
26. **What gets backed up:**
- `pg_dump -Fc` of each Postgres DB, run nightly **before** the restic job. Dumps are what we restore from. Copying the volume of a running DB isn't safe. Include the bots.ac app DB and its attachments.
- GlitchTip's Postgres, dumped the same way.
- `/captain/data` (CapRover config and app volumes) for Kuma, Readeck, Homepage config, Backrest config, Gotify/ntfy and Beszel.
- `/etc` (ufw, ssh, samba, cockpit).
- The Samba share folders (**not** `/srv/timemachine`).
- Syncthing config. The synced data only if the Mac isn't the main copy.
- **Always keep** the Grok Bot box copy at `~/backups/grokbot` on the server. That copy is never dropped.

**Excluded:** Docker images, caches, `/var/lib/docker/overlay2`, and logs. **Dropped:** R2 / age-to-R2 as a backup target (Decision Dani 2026-10-16).
27. **Off the box = rsync to Dani's Mac, then iCloud (decision 2026-10-16; replaces Backblaze B2 and R2).** Backrest/restic writes an encrypted local repo on the server (e.g. `/srv/restic-repo`). The Mac then **pulls** that repo with rsync over SSH across Tailscale into a folder inside iCloud Drive. iCloud carries it off-site, and Time Machine also versions it.
- **27a. Copy the restic repo only.** Never rsync live Postgres files. Copy only the encrypted restic repo, which holds the `pg_dump` files (rule 26). Restic encrypts everything except credentials, and the repo is just a directory of files ([restic docs](https://restic.readthedocs.io/en/stable/030_preparing_a_new_repo.html)). So the copy on Apple's servers is ciphertext.
- **27b. Direction: the Mac pulls (★).** A launchd job on the Mac runs `rsync -a --partial` from `bot-backup@<tailnet-name>:` into the iCloud folder. On the server, that Mac key's `authorized_keys` line is `command="rrsync -ro /srv/restic-repo",restrict,from="100.64.0.0/10" ssh-ed25519 …`. This makes the key read-only, limited to that one folder, and rsync-only ([rrsync(1)](https://man7.org/linux/man-pages/man1/rrsync.1.html), [serverfault](https://serverfault.com/questions/965053/restricting-a-ssh-key-to-only-allow-rsync-file-transfer)). The server holds **no** credentials for the Mac, so a hacked server can't wipe the Mac copy. The fallback is the server pushing to the Mac's Remote Login with `rrsync -wo -no-del` on the Mac key. That needs SSH open on the Mac, so it's second choice. See Q-B.
- **27c. Timing.** Pull at **05:00 ET**, after the 03:30 restic job and the weekly `check`. Don't pull while a backup or prune is running: either check for restic's lock first, or skip if the Backrest job is unfinished. restic writes new pack files and doesn't edit old ones, so incremental rsync stays small. That's my inference from how the repo is laid out, not something I checked in the docs. Use `--delete` only so that `prune` can free space. Time Machine (27f) keeps older versions in case a bad sync deletes files.
- **27d. The Mac must be awake.** Use `sudo pmset repeat wakeorpoweron MTWRFSU 04:55:00` and a launchd `StartCalendarInterval` job at 05:00. Wrap the job in `caffeinate -i` ([Apple pmset](https://support.apple.com/guide/mac-help/schedule-your-mac-to-turn-on-or-off-mchl40376151/mac), [napless guide](https://napless.io/guides/schedule-mac-wake/)). If the Mac was asleep, launchd runs the job at the next wake. If it was **off**, the job doesn't run until the next scheduled time ([Apple launchd docs](https://developer.apple.com/library/archive/documentation/MacOSX/Conceptual/BPSystemStartup/Chapters/ScheduledJobs.html)). A closed MacBook running on battery may not wake. I haven't verified this, so plan to keep it on power at night. Don't count on Power Nap to run this job. I found no Apple source saying it runs third-party launchd jobs.
- **27e. Missed-backup alert.** When the job succeeds, it pings a Healthchecks.io check from the Mac (same service as rule 41). The check's period is 1 day with a 12 h grace, so a sleeping, offline or broken Mac alerts Dani's phone. The server's Kuma also gets a push monitor: a successful pull writes a stamp file over the same rrsync session, or Kuma watches the Healthchecks badge. Backrest's own alert (rule 29) still covers the server-side job.
- **27f. iCloud folder rules.** The destination is `~/Library/Mobile Documents/com~apple~CloudDocs/bots-ac-backup/restic-repo` (iCloud Drive). **Turn off "Optimize Mac Storage"**, or right-click the folder → **Keep Downloaded** (macOS 15+). If you don't, macOS can evict files to placeholders, and rsync/Time Machine then see "dataless" files ([Apple](https://support.apple.com/guide/mac-help/optimize-storage-space-sysp4ee93ca4/mac), [Macworld](https://www.macworld.com/article/2478934/how-to-keep-an-icloud-drive-file-on-your-mac-while-optimizing-storage.html), [Eclectic Light](https://eclecticlight.co/2024/03/11/icloud-drive-in-sonoma-optimise-mac-storage-or-not/)). The repo must fit on **both** the Mac's free disk and the iCloud plan. Plans are 50 GB $0.99, 200 GB $2.99, 2 TB $9.99, 6 TB $29.99 and 12 TB $59.99 per month (US, [apple.com](https://support.apple.com/en-us/108047)). The server's `~/backups` is already 74 GB, and the restic repo size isn't known yet. Measure it before you pick a plan. Check daily that iCloud has finished uploading (no cloud-with-arrow badges). I haven't found a CLI to check this, so it's manual for now.
- **27g. Time Machine also covers this folder.** Don't add it to Time Machine exclusions. That gives point-in-time versions of the Mac copy, which protects against an rsync `--delete` mistake or iCloud sync damage. **Caveat:** the Mac's Time Machine target is bots.ac itself (rule 31b). So this layer only protects against Mac-side mistakes, not against losing the server. The off-site copy is iCloud.
- **27h. Optional second copy:** a spare RackNerd box over SFTP (restic `copy` or a second repo). See Q-C.
- **27i. Not chosen: rclone's `iclouddrive` backend from the server.** restic and Backrest *can* use it (`rclone:remote:path`, [restic](https://restic.readthedocs.io/en/stable/030_preparing_a_new_repo.html), [Backrest](https://garethgeorge.github.io/backrest/introduction/getting-started)). But it needs your real Apple ID password plus 2FA (app-specific passwords aren't accepted), and the trust token **expires every 30 days**, after which you have to re-auth by hand with 2FA ([rclone docs](https://rclone.org/iclouddrive/)). The code returns "trust token expired, please reauth" rather than recovering ([client.go](https://github.com/rclone/rclone/blob/master/backend/iclouddrive/api/client.go)). Advanced Data Protection is now documented as supported, *but* "Access iCloud Data on the Web" must be on, and users report hard `requestPCS` failures ([rclone docs](https://rclone.org/iclouddrive/), [issue #9658](https://github.com/rclone/rclone/issues/9658)). It uses Apple's web-login flow. Apple publishes no Linux API, and I found no sign Apple supports this. It would also put your Apple ID password on the server. **Too fragile for unattended nightly jobs.**
- The restic repo password lives in **1Password** only. If it's lost, every copy is lost.
28. **Schedule and retention:** Run nightly at 03:30 ET. Keep 7 daily, 4 weekly and 6 monthly. Run `restic check` weekly and `--read-data-subset=5%` monthly.
29. **Restore tests:** Once a month, restore the latest Postgres dump into a throwaway container and run a row-count check. Every quarter, restore one full app volume onto a spare server. A backup that hasn't been restored doesn't count. Backrest sends a failed or missed backup alert to ntfy/Gotify. Kuma gets a "heartbeat" push monitor that alerts if no success ping arrives within 26 hours.

## 7. App-specific edge cases
30. **Syncthing:** Turn off global discovery, relays and NAT-PMP/UPnP (`relaysEnabled=false`, `globalAnnounceEnabled=false`, `natEnabled=false`). Set the Mac's device address to `tcp://<tailnet-ip>:<syncthing-port>`. Bind the GUI to 127.0.0.1 or the tailnet IP, with a GUI password set. That keeps traffic off public relays and stops the box announcing itself. In CapRover, remove the template's public port mapping, or bind it to the tailnet IP.
31. **Samba + Mac:** Turn on `vfs objects = catia fruit streams_xattr`, `fruit:metadata = stream`, `fruit:model = MacSamba`, `fruit:posix_rename = yes`, `fruit:zero_file_id = yes`, and `server min protocol = SMB3`. Use a separate Samba user (`smbpasswd`) whose password is in 1Password. Connect with `smb://<tailnet-ip or MagicDNS name>`. On macOS, `.DS_Store` writes can be slow, so set `veto files` or `defaults write com.apple.desktopservices DSDontWriteNetworkStores true`.
31b. **Time Machine share (decision 2026-10-16).**
- **Separate share, nothing else stored on it.** `[timemachine]` with `path = /srv/timemachine`, `vfs objects = catia fruit streams_xattr`, `fruit:time machine = yes`, `fruit:time machine max size = 200G` (see Q-A), `valid users = tm-dani`, `read only = no`, `browseable = yes`. `fruit:time machine = yes` advertises FULLSYNC and forces `durable handles = yes`, `kernel oplocks = no`, `kernel share modes = no` and `posix locking = no`. The max size is only an approximation, based on the sparsebundle's band count, so **you must not store other files on this share** ([vfs_fruit(8)](https://www.samba.org/samba/docs/current/man-html/vfs_fruit.8.html)). Put fruit on every share, because "mixing shares with and without fruit will globally disable AAPL" (same source).
- **SMB3 only.** Keep `server min protocol = SMB3` (rule 31). Apple supports NAS Time Machine over SMB, and AFP isn't supported in macOS 27 ([Apple](https://support.apple.com/en-us/102423)).
- **Add it by hand, not via Bonjour.** Tailscale doesn't forward mDNS/Bonjour, so add the share with its tailnet name every time: `sudo tmutil setdestination -a -p smb://tm-dani@<magicdns-name>/timemachine`. Then run `sudo tmutil enable` and check with `tmutil destinationinfo` ([Tailscale TW guide](https://tailscale.tw/use-cases/time-machine-over-tailscale/), [tmutil(8)](https://keith.github.io/xcode-man-pages/tmutil.8.html), [Umbrel thread](https://community.umbrel.com/t/time-machine-backup-problem/24265/14)). A manual `tmutil` destination works across subnets and VPNs ([dwood.dev](https://dwood.dev/posts/create-samba-time-machine/)). **Skip Avahi.** It only helps on a LAN, and the box is a remote VPS. If it's ever wanted, set `multicast dns register = no` and add an `_adisk._tcp` service file ([jhnr.ch](https://blog.jhnr.ch/2023/01/09/setup-apple-time-machine-network-drive-with-samba-on-ubuntu-22.04/)). Avahi must never listen on the public interface.
- **Turn on encryption** when adding the disk in System Settings. It can't be added later without starting a fresh backup ([Umbrel thread](https://community.umbrel.com/t/time-machine-backup-problem/24265/14)). The key goes in 1Password.
- **Corruption risk.** Frequent disconnects during a backup make "Time Machine needs to create a new backup" more likely, so keep another copy if it matters ([Tailscale TW](https://tailscale.tw/use-cases/time-machine-over-tailscale/)). Do the first full backup on a stable, fast connection. If Time Machine asks to start over, run the full backup again. Don't try to repair it.
- **Not a server backup.** This protects the **Mac**. It is not an off-site copy of bots.ac. If the server dies, the Mac's Time Machine history goes with it. The Mac's own off-site copy stays iCloud.
- **Disk:** 380 GB free. A 200G cap leaves about 180 GB for the restic repo, Docker and growth. Beszel's 80% disk alert (rule 33) counts Time Machine usage. Back up the Time Machine share with Backrest? **No.** Exclude `/srv/timemachine` from rule 26, because it's already a backup and it's encrypted.
32. **Let's Encrypt limits** (about 50 certs per domain per week, 5 identical certs per week, and 5 failed checks per hostname per hour): DNS must point at the box **before** you click "Enable HTTPS". Turn on one app at a time. If it fails, wait an hour and don't retry over and over. Don't delete and recreate apps just to "fix" certs.
33. **Disk filling up:**
- Docker log rotation in `/etc/docker/daemon.json`: `"log-driver":"json-file","log-opts":{"max-size":"10m","max-file":"3"}`.
- journald: `SystemMaxUse=1G`.
- Kuma: set history retention to **30 days** (default 180).
- GlitchTip: set event retention to 30 days.
- Weekly `docker image prune` (dangling images only, never `-a` without asking).
- Beszel alerts at 80% disk, critical at 90%.
34. **Time:** Server clock and logs stay in **UTC** (backups and logs stay simple, with no daylight-saving gaps). Schedules get written with the ET time noted next to them. `systemd-timesyncd` stays on. Check it with `timedatectl` ("System clock synchronized: yes"). Let's Encrypt, TOTP and restic all break if the clock drifts.

## 8. Updates
35. **unattended-upgrades for security only** (`${distro_id}:${distro_codename}-security`, plus ESM if it's enabled). **Docker and CapRover packages are on a blacklist** (`docker-ce*`, `containerd.io`). We upgrade them by hand after reading the release notes and making a backup.
36. **Reboot window:** Set `Automatic-Reboot "true"` with `Automatic-Reboot-Time "09:00"` UTC (**5:00 AM ET**). Kuma's external check (rule 41) must show the box back up within 10 minutes. After any reboot, check rules 8 and 23 (DOCKER-USER still in place).
37. **CapRover and app images:** update by hand monthly, one app at a time, with the image version pinned (no `:latest` for stateful apps).

## 9. Resource limits
38. **Every CapRover app gets a memory cap** (CapRover's Service Update Override `Resources.Limits.MemoryBytes`). Starting caps:
- Kuma 256M
- Beszel hub 192M
- Backrest 512M
- Gotify/ntfy 128M
- GlitchTip web 768M + worker 512M + Redis 256M
- Readeck 512M
- Homepage 256M
- Syncthing 512M

Leave about 4 GiB for the OS, Postgres and Samba.
39. **Swap:** add a **4 GiB swapfile** with `vm.swappiness=10`, so a spike slows things down instead of the kernel's OOM killer killing Postgres. Protect Postgres and sshd with `OOMScoreAdjust=-500`.
40. **Alerts:** Beszel alerts when RAM is over 85% for 10 minutes, swap use is over 1 GiB, CPU is over 90% for 15 minutes, or disk is over 80%. Kuma watches every app's URL. Both send to ntfy/Gotify.

## 10. Monitoring
41. **The box can't report its own death.** Add one **outside** check: a free external monitor (Healthchecks.io, UptimeRobot's free tier, or Kuma on a spare RackNerd box; see Q5) that pings `https://bots.ac` and the SSH port, and alerts Dani's phone by email or push. This must **not** depend on ntfy running on bots.ac.
42. Kuma on bots.ac also sends a heartbeat *out* to that external monitor every 5 minutes. If the heartbeat stops, Dani gets an alert.

## 11. Secrets
43. **All secrets live in 1Password** (a "bots.ac" vault), including:
- the CapRover password and OTP seed
- the restic repo password, the Mac pull key passphrase, the Time Machine encryption key and the `tm-dani` Samba password
- Postgres passwords
- the Samba password
- Tailscale auth keys
- the Gotify/ntfy tokens
- the GlitchTip secret key
- the console password

They go into CapRover as **app environment variables** by hand or with `op` CLI. They never go into git repos, `captain-definition` files, the change log, or chat. Add `gitleaks` as a pre-commit hook in any ops repo.

## 12. Install and change order (each step has a check and a rollback)
| # | Step | Check it worked | Rollback |
|---|---|---|---|
| 1 | Read-only audit: `ss -tulpn`, `ufw status`, `ip6tables -S`, Postgres listen address, `timedatectl`, disk, Servarica console login works | List of open ports saved | n/a |
| 2 | First full backup of audiovault + Postgres (`pg_dump`) copied off the box | Restore it into a temp DB | n/a |
| 3 | Tailscale install + join, key expiry turned off | Ping from Mac on 100.x | `tailscale down` |
| 4 | SSH hardening (rules 16–19) with the lockout routine | Fresh login on <ssh-port> with key; password refused | Undo the sshd override from a still-open session or the console |
| 5 | UFW on (rule 6–7, 10), with `at` auto-undo | nmap from outside | Auto `ufw disable` |
| 6 | Lock Postgres to localhost plus the tailnet IP (rule 24), after a fresh `pg_dump` | nmap shows <pg-port> closed; app still works over localhost/tailnet | Restore old `postgresql.conf` / `pg_hba.conf` |
| 7 | Swap, journald cap, timesyncd, unattended-upgrades | `swapon`, `timedatectl` | Remove the files |
| 8 | Install Docker + daemon.json log limits | `docker info` | `apt remove docker-ce` |
| 9 | DOCKER-USER / ufw-docker rules | `iptables -L DOCKER-USER` | Restore the saved rules |
| 10 | Install CapRover → change `captain42` → 2FA → HTTPS + Force HTTPS → block <setup-port> → dashboard allowlist (tailnet + GitHub Actions IPs) | Dashboard only at captain.dev.bots.ac from the tailnet or CI ranges; nmap clean | Stop/remove the captain container (apps not deployed yet) |
| 11 | Gotify/ntfy, then Kuma (with the external check), then Beszel | Test alert arrives on phone | Remove the app |
| 12 | Backrest + nightly jobs + **first restore test** | Restore works | Remove the app |
| 13 | Homepage, Readeck, GlitchTip (one per day) | Memory under the cap 24 h later | Remove the app |
| 14 | Syncthing (rule 30), Cockpit, Samba (rules 12, 31, 31b) | Mac connects over the tailnet; first Time Machine backup completes; nmap clean | Stop the service |
| 15 | Mac pull job (rule 27): rrsync key, launchd + pmset, Healthchecks | Healthchecks check green 2 nights running; `restic check` against the Mac copy | Remove key line + launchd plist |
After each step: run nmap from outside, add a change-log entry, and wait a day before big steps.

## 13. Premortem ("it's 3 months later and it broke. Why?")
- Docker upgrade reset iptables and port <setup-port> or <pg-port> was open. **Covered by** rules 8, 23 and 36 (checking after reboots and upgrades).
- Tailscale broke and Dani was locked out. **Covered by** rule 14 (public key-only SSH plus the console).
- A bot ran `docker system prune -a --volumes`. **Covered by** rules 2–3 (no sudo for it, needs a second yes, backup first).
- Backups ran for months but couldn't be restored. **Covered by** rule 29.
- Disk filled up from Kuma, logs or GlitchTip. **Covered by** rule 33.
- RAM ran out and the kernel killed Postgres. **Covered by** rules 38–39.
- IPv6 left ports open that IPv4 blocked. **Covered by** rule 10.
- Syncthing leaked through public relays. **Covered by** rule 30.
- Let's Encrypt blocked new certs. **Covered by** rule 32.
- The Mac slept or travelled for a week, so off-site backups silently stopped. **Covered by** rules 27d–27e.
- iCloud evicted repo files, or the iCloud plan filled up, so the off-site copy was incomplete. **Covered by** rule 27f.
- Time Machine filled the server disk. **Covered by** rule 31b (the cap).

## 14. Assumptions (TypeSafe-style confidence)
| # | Assumption | Confidence |
|---|---|---|
| A1 | Server facts (Ubuntu 24.04, 16 vCPU, 15 GiB, no Docker/CapRover yet) are correct as of the 10-13 audit | 90% |
| A2 | Legacy Postgres is native (apt), not in Docker | 60% |
| A3 | Postgres <pg-port> might be open publicly right now (unverified); if open, close first after a fresh dump (Q1) | 50% (open status unverified) |
| A4 | Servarica gives a web VNC/console login (chosen as lockout fallback, Q3) | **100% / confirmed** |
| A5 | Dani will make a console password for `dani` and keep it in 1Password (Q3) | **100% / confirmed** |
| A6 | Keep IPv6 on and mirror every firewall rule in ip6tables (Q7); whether the box already has a public IPv6 address is still unverified | 65% (address); policy **100% / confirmed** |
| A7 | DNS keeps `*.bots.ac`, plus per-app and apex records; CapRover catch-all returns 404 (premortem Q2, decided) | **100% / decided** |
| A8 | CapRover Pro includes 2FA for the dashboard | 80% |
| A9 | Off-box copy = Mac pull + iCloud (Dani's decision); B2 dropped (Q-B) | **100% / confirmed** |
| A17 | Dani's MacBook is plugged in and asleep (not shut down) most nights, so the 05:00 wake works (Q-E) | **100% / confirmed** |
| A18 | The restic repo (today ~74 GB in `~/backups`, final size unknown) fits on the Mac's free disk and Dani's **iCloud 2 TB** plan | **100% / confirmed** (plan size); repo fit on disk still measure at install |
| A19 | **Dani's iCloud plan: 2 TB** (already subscribed; no upgrade needed, Q-D) | **100% / confirmed** |
| A20 | Time Machine cap is **200G** (Q-A / Q8) | **100% / confirmed** |
| A21 | Advanced Data Protection state doesn't matter (data is restic-encrypted and the Mac's iCloud client handles upload) | 85% |
| A22 | The tailnet is stable enough for Time Machine; disconnects can still force a fresh backup | 70% |
| A10 | A spare RackNerd box gets a weekly restic SFTP second copy (Q-C); external uptime uses Healthchecks.io/UptimeRobot (Q5) | **100% / confirmed** |
| A11 | Dani's Mac is the main copy of Syncthing data; server still backs it up as insurance (Q13) | **100% / confirmed** |
| A12 | Phones need push alerts while off the tailnet via ntfy public endpoint + auth tokens (Q9) | **100% / confirmed** |
| A13 | Single-node CapRover (no extra swarm nodes) | 90% |
| A14 | Bots get SSH access through their own `bot` user, own key, read-only sudo (Q2) | **100% / confirmed** |
| A15 | UTC on the server is fine; ET in reports (Q11) | **100% / confirmed** |
| A16 | Memory caps above are good starting points (not measured) | 70% |

## 15. Questions for Dani (★ = recommended) — all ANSWERED 2026-10-16 4:53 AM ET ("Yes, use all recommended")
- **Q1 (A3, security/data):** **ANSWERED ★** Close Postgres <pg-port> first, before anything else, after a fresh dump. (Was: a ★ / b CapRover install / c Leave it)
- **Q2 (A14, security):** **ANSWERED ★** A `bot` user, own key, sudo only for read-only commands. (Was: a ★ / b Dani's account / c No bot SSH)
- **Q3 (A4/A5, security):** **ANSWERED ★** Public key-only SSH on <ssh-port> + Servarica console + a console password in 1Password. (Was: a ★ / b Tailscale-only / c Console only)
- **Q4 (security):** **ANSWERED ★** fail2ban (simple, local). (Was: a ★ / b CrowdSec / c Neither)
- **Q5 (A10, public/money):** **ANSWERED ★** Free Healthchecks.io/UptimeRobot plus heartbeat. (Was: a ★ / b Second Kuma on RackNerd / c Both)
- **Q6:** **ANSWERED** 2026-10-16: Mac + iCloud. See Q-B.
- **Q7 (A6, security):** **ANSWERED ★** Keep IPv6; mirror every rule in ip6tables and test. (Was: a ★ / b Turn off / c Ignore)
- **Q8:** **ANSWERED** 2026-10-16: Samba **does** serve Time Machine, with a **200G** cap (already decided; see Q-A / rule 31b).
- **Q9 (A12, public):** **ANSWERED ★** ntfy public endpoint with auth tokens; admin private. (Was: a ★ / b Tailscale on phone only / c Gotify public)
- **Q10 (public):** **ANSWERED ★** Only sites, the Kuma status page and error ingest; all admin UIs tailnet-only. (Was: a ★ / b Admin UIs public with Basic Auth / c Everything public)
- **Q11 (A15):** **ANSWERED ★** UTC on the server; ET in reports. (Was: a ★ / b America/New_York on the server)
- **Q12 (data loss):** **ANSWERED ★** Yes, unattended reboots — security-only, 5:00 AM ET. (Was: a ★ / b Patch but reboot by hand / c No auto updates)
- **Q13 (A11, data loss):** **ANSWERED ★** Yes, back up Syncthing data on the server (cheap insurance). (Was: a ★ / b No, Mac is enough)
- **Q14:** **ANSWERED** Replaced by Q-D (see below).
- **Q-A (A20, disk):** **ANSWERED ★** Time Machine cap **200G** (leaves ~180 GB for the restic repo, Docker and growth). (Was: a 150G / b ★ 200G / c 250G / d Size at 1.5× after measuring)
- **Q-B (A9/A17, data loss):** **ANSWERED ★** Mac **pulls** the restic repo nightly over Tailscale with rsync (read-only `rrsync` key, launchd + pmset wake, Healthchecks) into an iCloud Drive folder that Time Machine also covers. (Was: a ★ / b Server push / c Syncthing / d rclone iclouddrive)
- **Q-C (A10, data loss/money):** **ANSWERED ★** Yes — second copy on spare RackNerd via restic over SFTP weekly. (Was: a ★ / b No / c Later)
- **Q-D (A18/A19, money):** **ANSWERED** Dani **already has the iCloud 2 TB plan**; no upgrade is needed. (Recommended was measure-first / stay / go to 2 TB now — exception: existing 2 TB subscription.)
- **Q-E (A17):** **ANSWERED ★** Plugged in, lid closed or open, asleep (wake at 04:55 works). (Was: a ★ / b Usually shut down / c Often travelling)
