# bots.ac server plans: pre-mortem

Date: 2026-10-16. Owner bot: bots.ac. Status: review only; decisions 1-6 applied 2026-10-16. Nothing on the server was touched. Questions 7-20 remain open.

Reviewed: the CapRover fresh-install plan (Phase 0 done), its security audit, the 2026-10-13 health audit, Linux Doctor output and inventory, the bots.ac operating rules (accepted 2026-10-16), and the bots.ac DESIGN and PLAN. The Ansvisor research is out of scope (owned by Personal Brand).

Public repo note: this repo is public, so every IP address, the SSH port and other host-specific port numbers are replaced with placeholders such as `<server-ip>` and `<ssh-port>` in this folder. Well-known web ports (80, 443) are kept.

## The scenario

It is mid-January 2027. The bots.ac setup failed badly. Below is how it most likely happened, worst first.

Likelihood and impact are High, Medium or Low. "Fix goes in" names the plan step that should own the fix. "Install plan" is `2026-10-13-caprover-fresh-install-plan.md`, "Rules" is `2026-10-16-bots-ac-rules.md`, "App PLAN" is the bots.ac `PLAN.md`.

## Failure stories

### F1. Dani got locked out of SSH because the rules use the wrong port
- What happened: the server listens on SSH port A (install plan, fail2ban jail, ufw allow rule). The accepted rules say port B everywhere (rules 6, 11, 16, 19, Q3, step 4). The two numbers differ by one digit. An executor following the rules set ufw to allow only port B and moved `ssh.socket` to B, while fail2ban and the old allow rule still pointed at A. Then the auto-undo timer was forgotten.
- Likelihood: High (the rules are the newest doc and were accepted wholesale). Impact: High (console-only recovery).
- Early warning: any plan or command that mentions port B; `ss -tlnp` and `ufw status` disagree.
- Fix: decide one port, then correct every doc before any rules step runs. Recommended: keep the current port A, which already works and is hardened. Fix goes in: Rules section 1 and 3 and step 4 (edit), install plan Global constraints (confirm).
- Decision (Dani 2026-10-16): SSH stays on the current port; fix the rules so they use the same port (the rules wrongly had a one-digit-different value).

### F2. The rules redid Phase 0 on a live host and broke it
- What happened: the rules' step table assumes a fresh server (install Tailscale, harden SSH, turn on ufw). Phase 0 already did those. Redoing step 5 (`ufw` with its own rule list) dropped the Tailscale UDP rule and the explicit swarm denies; redoing the Tailscale join re-keyed the node and changed its tailnet IP, which broke the fail2ban ignore list and Postgres binding.
- Likelihood: Medium. Impact: High.
- Early warning: an executor asks to run rules step 3, 4 or 5.
- Fix: mark rules steps 3 to 5 and the journald cap as "done in install plan Phase 0, verify only". Fix goes in: Rules section 12 (step table).

### F3. CI deploys stopped the day the dashboard went tailnet-only
- What happened: the rules (rule 13, 20, Q10) put the CapRover dashboard behind an nginx tailnet-only allowlist. GitHub Actions deploys (`caprover/deploy-from-github`) call the same captain hostname from GitHub's runners, which are not on the tailnet, so every deploy got 403. Someone "temporarily" removed the allowlist and never put it back.
- Likelihood: High. Impact: Medium (deploys broken, then dashboard public).
- Early warning: the first deploy after install plan Task 5 Step 4 fails with 403.
- Fix: allow GitHub Actions IP ranges from `api.github.com/meta` (auto-refreshed) on the captain nginx allowlist, alongside the tailnet. Residual risk: those ranges are shared by all GitHub users, so protection rests on the CapRover password, 2FA and a deploy-only token. Fix goes in: install plan Task 5 Step 4 and Task 6 Step 3.
- Decision (Dani 2026-10-16): Allow GitHub Actions IP ranges (`api.github.com/meta`, auto-refreshed). Residual risk accepted as above.

### F4. A Docker or CapRover update reopened the setup port or the Docker socket path
- What happened: two firewall mechanisms were in play, the install plan's `fw-caprover.service` and the rules' `ufw-docker` entries in `after.rules`. They disagreed after a reload; the systemd unit was removed as a "duplicate", then a Docker upgrade flushed DOCKER-USER and the CapRover setup port was public again for weeks. IPv6 had no matching DOCKER-USER rule at all.
- Likelihood: Medium. Impact: High (dashboard on the setup port is root on the host).
- Early warning: `iptables -S DOCKER-USER` differs from the doc; no alert fires because the check in rule 23 was never built.
- Fix: pick one mechanism, mirror it in ip6tables, and add a timer that alerts if the drop rule is missing. Fix goes in: install plan Task 3 Step 2, Rules rule 8, 10 and 23.

### F5. Disk filled up and Postgres went read-only
- What happened: 380 GB free, minus a 200 GB Time Machine share, minus a restic repo (the old `~/backups` is already about 74 GB), minus Docker images, OpenSandbox machine images (Chromium), build cache from on-server builds, the shared browser-profile volume and two Postgres clusters. The 80% alert went to ntfy on the same host, which was itself down.
- Likelihood: Medium. Impact: High.
- Early warning: disk over 60% within the first month; Docker build cache over 20 GB.
- Fix: a written disk budget, `docker builder prune` weekly, Time Machine cap sized after measuring, and disk alerts that go through the external monitor. Fix goes in: Rules rule 33 and 31b, App PLAN Task 0.2.

### F6. RAM ran out when OpenSandbox machines and builds ran together
- What happened: 15 GiB, no swap yet. The rules' memory caps total about 4.5 GiB for small apps, but the bots.ac orchestrator, a CapRover Postgres 17, three to ten OpenSandbox Chromium machines and an on-server image build were not budgeted. The kernel killed Postgres 16 (audiovault) and the CapRover nginx.
- Likelihood: Medium. Impact: High.
- Early warning: swap use over 1 GiB, OOM lines in `dmesg`, Beszel RAM alerts.
- Fix: build images in GitHub Actions (install plan Task 6), never on the server; cap local OpenSandbox concurrency at 2 and send the rest to Agent 37; add swap before Docker. Fix goes in: Rules step 7 (swap) before step 8, App PLAN Task 0.2 and 1.5.4.

### F7. Backups existed in three places and none restored
- What happened: three backup designs coexist. The install plan pulls age-encrypted tars to the shared Grok Bot box weekly. The rules use restic plus a Mac pull into iCloud plus a RackNerd copy. DESIGN uses age to R2 with 30/12 retention. Each was half built. The restic password and age key were in different vaults (Nebula vs a "bots.ac" vault). When the disk died, the newest restorable Postgres dump was weeks old.
- Likelihood: High. Impact: High (data loss).
- Early warning: no restore drill on the calendar after Task 7; Healthchecks for the Mac pull is yellow.
- Fix: one backup design: the rules' restic design, with the bots.ac app DB and attachments added; drop R2. The Grok Bot box copy (`~/backups/grokbot` on the server) is always kept, never dropped. One key location, and the monthly restore drill. Fix goes in: Rules section 6, install plan Task 7, App PLAN Task 0.5.
- Decision (Dani 2026-10-16): Restic design plus the bots.ac DB; drop R2; always keep `~/backups/grokbot` on the server.

### F8. Schedules drifted by an hour on November 1 and reboots hit the backup
- What happened: the server runs on UTC (rule 34). Schedules were written in ET and converted once, during daylight time. Daylight time ends 2026-11-01. The unattended reboot at 09:00 UTC is 5:00 AM EDT but 4:00 AM EST. The restic job at 03:30 ET, the weekly `check` and the Mac pull at 05:00 ET all moved relative to each other, and the reboot interrupted a prune, leaving a stale restic lock. The Mac (on ET) pulled a half-written repo.
- Likelihood: High. Impact: Medium.
- Early warning: restic "repository is already locked" errors after 2026-11-01.
- Fix: write server timers in UTC with fixed gaps (backup 06:30 UTC, check after, reboot no earlier than 3 hours after the backup), or use systemd timers with `America/New_York` in `OnCalendar`. Make the reboot skip if restic holds a lock. Fix goes in: Rules rule 28, 34 and 36.

### F9. The *.bots.ac wildcard exposed things and confused production DNS
- What happened: production was supposed to get one DNS record per app, but a `*.bots.ac` wildcard was also created. Every typo and every unused name resolved to the server; CapRover's catch-all page answered. Old preview names stayed reachable. When the server was later moved, the wildcard still pointed at the released IP (subdomain takeover). The apex `bots.ac` is not covered by a wildcard at all, so the main app had no record.
- Likelihood: Medium. Impact: Medium (public exposure).
- Early warning: `dig random123.bots.ac` returns the server IP.
- Fix: keep `*.bots.ac`, and also add per-app records plus an apex record for `bots.ac`. Guard: CapRover's catch-all must return 404 for unknown hostnames. Keep `*.dev.bots.ac` for staging. Fix goes in: install plan Task 1 Step 2, Rules rule 22, and the Decommission checklist.
- Decision (Dani 2026-10-16): Keep the `*.bots.ac` wildcard and also add per-app records plus an apex record; CapRover catch-all returns 404 for unknown hostnames.

### F10. The dashboard hostname in the docs did not match the real one
- What happened: the install plan uses `captain.dev.bots.ac` (root domain `dev.bots.ac`). The rules use `captain.bots.ac` and a `*.bots.ac` wildcard (rules 20, 22, step 10). The allowlist and alerts were set up for the wrong host.
- Likelihood: Medium. Impact: Low to Medium.
- Early warning: a doc or command that says `captain.bots.ac`.
- Fix: dashboard is `captain.dev.bots.ac` (replace `captain.bots.ac` in the rules). Root domain stays `dev.bots.ac`; production apps get custom domains. Fix goes in: Rules rule 20 and 22, step 10.
- Decision (Dani 2026-10-16): Dashboard hostname is `captain.dev.bots.ac`.

### F11. A bot with Dani's account and passwordless sudo did damage
- What happened: rule 3 and Q2 say bots use a `bot` user with read-only sudo. The install plan still runs as Dani, and the `sudo` group has NOPASSWD for everything (health item 5, still open). A bot ran a "cleanup" prune while following a stale plan.
- Likelihood: Medium. Impact: High.
- Early warning: executor commands start with `ssh manage.bots.ac 'sudo ...'` under Dani's user.
- Fix: create the `bot` user and remove group-wide NOPASSWD before CapRover install. Fix goes in: new install plan Task 0.8 (gated), Rules rule 3.

### F12. Postgres stayed reachable in ways nobody intended
- What happened: the brief says Postgres must be reachable only over Tailscale. Rule 24 says listen on 127.0.0.1 only. The health audit suggests localhost plus the tailnet IP. The install plan says Postgres is out of scope. Today ufw blocks the port publicly, but Postgres still listens on all addresses and pg_hba still allows `0.0.0.0/0` and `::/0`. One `ufw disable` during a rollback reopened it.
- Likelihood: Medium. Impact: High.
- Early warning: `ss -tlnp` still shows Postgres on all addresses.
- Fix: listen on localhost plus the tailnet IP, pg_hba limited to the tailnet range and localhost, keep the ufw deny as a second layer. Fresh dump first. Fix goes in: Rules step 6 (Q1), as the first change after the current state check.
- Decision (Dani 2026-10-16): Postgres listens on localhost plus the tailnet IP only, after a fresh `pg_dump`.

### F13. OpenSandbox became the way in
- What happened: OpenSandbox shares the host with Postgres, the CapRover dashboard and Time Machine. A guest or student task ran in a local sandbox; the socket proxy allowed `create` with host mounts, so a container mounted `/`. DESIGN also says bots.ac is "not exposed publicly" in one line while the rest assumes a public `https://bots.ac` with share links.
- Likelihood: Low to Medium. Impact: High.
- Early warning: the socket-proxy test in App PLAN Task 0.2 only checks images/volumes/networks, not bind mounts or `--privileged`.
- Fix: deny bind mounts outside `STATE_ROOT`, privileged and host network in the proxy; send all untrusted work to Agent 37 from day one. Fix goes in: App PLAN Task 0.2 and 1.5.3, DESIGN section 3 (fix the wording).

### F14. Monitoring told Dani nothing
- What happened: alerts went to ntfy and Kuma on the same host. The external monitor (rule 41) was planned for step 11, after CapRover. The audiovault health check had already been failing since before October and nobody acted on it.
- Likelihood: Medium. Impact: Medium.
- Early warning: no external check exists on install day.
- Fix: set up the free external monitor before CapRover install. Fix goes in: Rules step 11, moved before step 10.

### F15. audiovau.lt broke and nobody had decided whether that was OK
- What happened: the install plan disables host nginx and certbot (api.audiovau.lt goes offline, accepted), and closing the API port cut public API clients. The rules list "real sites (audiovau.lt etc.)" as public. Users of the app noticed first.
- Likelihood: Medium. Impact: Medium.
- Fix: one written decision on audiovau.lt (retire it or migrate it into CapRover). Fix goes in: install plan Task 3 Step 3, Rules section 2.

### F16. Shared-box secrets and the CapRover Pro key leaked
- What happened: the 1Password service-account token file is readable by every agent on the shared Grok Bot box, and box-side backups land there too. A prompt-injected bot read it.
- Likelihood: Low. Impact: High.
- Fix: confirm the service account is read-only on one vault, rotate it on a schedule, and move backups off the shared box. Fix goes in: install plan "Gaps", Rules rule 43.

## Conflicts between the docs

| # | Topic | Doc A | Doc B | Recommended resolution |
|---|---|---|---|---|
| C1 | SSH port | Install plan and the live server: port A | Rules 6, 11, 16, 19, Q3, step 4: port B (one digit different) | Decision (Dani 2026-10-16): Keep A, fix the rules |
| C2 | Dashboard host | Install plan: `captain.dev.bots.ac`, root `dev.bots.ac` | Rules 20, 22, step 10: `captain.bots.ac`, wildcard `*.bots.ac` | Decision (Dani 2026-10-16): Keep `captain.dev.bots.ac` |
| C3 | Production DNS | Brief: one record per production app | A `*.bots.ac` wildcard was just created; rule 22 asks for it | Decision (Dani 2026-10-16): Keep the wildcard and also add per-app records plus an apex; CapRover catch-all returns 404 for unknown hostnames |
| C4 | Time zone | Rule 34: server on UTC | Rules 28, 27c, 36 and DESIGN 2.4 give ET times; reboot is pinned to 09:00 UTC "= 5:00 AM ET", true only during daylight time | UTC timers with fixed gaps, or timezone-aware timers |
| C5 | Postgres binding | Brief: Tailscale only. Health audit: localhost plus tailnet IP | Rule 24: 127.0.0.1 only. Install plan: out of scope | Decision (Dani 2026-10-16): Localhost plus the tailnet IP only, after a fresh `pg_dump` |
| C6 | Which Postgres | Host Postgres 16 (audiovault) | DESIGN: CapRover one-click Postgres 17 for bots.ac | Separate PG17 app is fine; budget RAM and back it up |
| C7 | Backups | Install plan Task 7: age to the shared box | Rules section 6: restic, Mac pull, iCloud, RackNerd. DESIGN: age to R2 | Decision (Dani 2026-10-16): Restic design plus the bots.ac DB; drop R2; always keep `~/backups/grokbot` on the server |
| C8 | Secrets vault | Install plan: vault "Nebula" | Rules 43 and PLAN A19: a "bots.ac" vault | Pick one |
| C9 | Image builds | Install plan: build in GitHub Actions, deploy by digest | DESIGN: built on the server from `captain-definition` | Build in Actions |
| C10 | Dashboard access | Install plan Task 5.4: box egress IP plus tailnet | Rules 13, 20: tailnet only. CI needs access too | Decision (Dani 2026-10-16): Allow GitHub Actions IP ranges (`api.github.com/meta`, auto-refreshed); residual risk rests on CapRover password, 2FA and a deploy-only token |
| C11 | Firewall for Docker | Install plan: `fw-caprover.service` | Rules 8: `ufw-docker` in `after.rules` | One mechanism, mirrored in IPv6 |
| C12 | Already done vs to do | Install plan Phase 0: SSH, fail2ban, ufw, journald 500M done | Rules steps 3 to 5 and 7 redo them; journald cap 1G | Mark as "verify only"; keep 500M |
| C13 | bots.ac exposure | DESIGN 1, 9: public `https://bots.ac` with share links | DESIGN 3 "Deployment": "not exposed publicly" | Public app behind passkeys; OpenSandbox private |
| C14 | audiovau.lt | Install plan: offline, accepted | Rules section 2: listed as a public real site | Decide once |
| C15 | Disk free | Install plan baseline: 451G | Brief and rules: 380 GB | Re-measure in the step 1 audit |
| C16 | Inventory data | The 2026-10-13 inventory's listener list includes processes (workerd, cursorsandbox, Paseo) that look like the Grok Bot box, and its DNS checks used the box's fake-IP resolver | Health audit uses external probes | Re-run the listener list on the server; check DNS through DoH only |

## Assumptions

| # | Assumption | Confidence |
|---|---|---|
| S1 | Phase 0 (SSH key-only, fail2ban, default-deny ufw, journald cap, updates) is done and still in place | 90% |
| S2 | The live SSH port is the install plan's value, and the rules' value is a typo | 85% |
| S3 | Docker and CapRover are not installed yet | 95% |
| S4 | The *.bots.ac wildcard stays; per-app and apex records are added; CapRover catch-all returns 404 for unknown hostnames (decided 2026-10-16) | **100% / decided** |
| S5 | Production apps will use custom domains on bots.ac while CapRover's root stays dev.bots.ac | 75% |
| S6 | CapRover Pro's 2FA works with the CLI and the deploy action for app-token deploys | 70% |
| S7 | CI reaches the dashboard via auto-refreshed GitHub Actions IP ranges from `api.github.com/meta` (decided 2026-10-16); residual risk accepted | **100% / decided** |
| S8 | 15 GiB RAM is enough for the small apps, bots.ac, PG17 and 2 local sandboxes, with swap | 60% |
| S9 | 380 GB is enough for a 200 GB Time Machine cap plus everything else for 3 months | 55% |
| S10 | Dani's Mac is awake and on power most nights for the 05:00 ET pull | 80% (Dani confirmed habit; travel not covered) |
| S11 | The tailnet IP of the server stays stable (no re-join) | 85% |
| S12 | The 1Password service account is read-only on one vault | 50% (not yet confirmed) |
| S13 | audiovau.lt can stay offline | 60% |
| S14 | Unattended security reboots do not break DOCKER-USER rules when the persistence unit exists | 75% |
| S15 | The OpenSandbox socket proxy can block bind mounts and privileged containers | 65% |
| S16 | Let's Encrypt limits are not a problem with at most 3 previews and per-app certs | 90% |
| S17 | Server IPv6 is either absent or covered by ufw with IPV6=yes | 65% |
| S18 | GHCR pulls with a read-only classic PAT on `droxey` are acceptable until a bot account exists | 70% |
| S19 | The unused RackNerd and LuxVPS servers can host the second backup copy and the outside nmap | 80% |
| S20 | Monthly CapRover/Docker updates by hand will actually happen | 50% |

## Questions for Dani

Each has a recommended answer marked ★. These cover every assumption under 90% plus anything touching money, security, data loss or public exposure.

1. **SSH port (security, lockout).** The rules say a different port than the server uses. a) ★ Keep the current port and fix the rules. b) Move to the rules' port with the full lockout routine. c) Something else.
   - Decision (Dani 2026-10-16): Keep the current port and fix the rules (the rules wrongly had a one-digit-different value).
2. **Production DNS (public exposure).** a) ★ Delete `*.bots.ac`; keep `*.dev.bots.ac`; add `bots.ac` and one record per production app. b) Keep `*.bots.ac` too. c) Use `*.bots.ac` instead of per-app records.
   - Decision (Dani 2026-10-16): Keep the `*.bots.ac` wildcard and also add per-app records plus an apex record; CapRover's catch-all returns 404 for unknown hostnames.
3. **Dashboard hostname.** a) ★ `captain.dev.bots.ac` (install plan). b) `captain.bots.ac` (rules).
   - Decision (Dani 2026-10-16): `captain.dev.bots.ac` (replace `captain.bots.ac`).
4. **CI deploys with a tailnet-only dashboard (security).** a) ★ GitHub Actions joins the tailnet for the deploy step with an ephemeral tagged key. b) Self-hosted runner on the tailnet. c) Allow GitHub's IP ranges (large, not recommended).
   - Decision (Dani 2026-10-16): Allow GitHub Actions IP ranges (`api.github.com/meta`, auto-refreshed). Residual risk: the ranges are shared by all GitHub users, so protection rests on the CapRover password, 2FA and a deploy-only token.
5. **Postgres binding (security).** a) ★ Localhost plus the tailnet IP, pg_hba limited to the tailnet and localhost, after a fresh dump. b) Localhost only (rule 24). c) Leave as is behind ufw.
   - Decision (Dani 2026-10-16): Localhost plus the tailnet IP only, after a fresh `pg_dump`.
6. **Backup design (data loss, money).** a) ★ The rules' restic design (Mac pull to iCloud plus RackNerd), with the bots.ac DB and attachments added; drop R2 and the box-side pulls. b) Keep R2 as well. c) Keep all three.
   - Decision (Dani 2026-10-16): The rules' restic design plus the bots.ac DB; drop R2. The Grok Bot box copy (`~/backups/grokbot` on the server) is always kept, never dropped.
7. **Secrets vault (security).** a) ★ One "bots.ac" vault for everything new, and move the CapRover and age items there. b) Keep Nebula.
8. **Schedules on a UTC server.** a) ★ Write timers in UTC with fixed gaps, and make the reboot wait for any restic lock. b) Timezone-aware timers in ET. c) Move the server to ET.
9. **Docker firewall mechanism (security).** a) ★ Keep the install plan's systemd unit, add IPv6 and a missing-rule alert. b) Switch to `ufw-docker` after.rules. 
10. **Bot access (security).** a) ★ Create the `bot` user and drop group-wide passwordless sudo before CapRover install. b) After install. c) Keep using Dani's account.
11. **Image builds (resources).** a) ★ Build in GitHub Actions only. b) Allow on-server `captain-definition` builds.
12. **OpenSandbox on this host (security, money).** a) ★ Local sandboxes only for Dani's own work, max 2 at once, bind mounts and privileged blocked; all untrusted work to Agent 37. b) Everything to Agent 37 (per-minute cost). c) Everything local.
13. **Time Machine on this server (disk, data loss).** a) ★ Measure the restic repo first, then set the cap (start at 150 GB). b) Keep 200 GB now. c) Put Time Machine elsewhere.
14. **Memory budget (stability).** a) ★ Add 4 GiB swap before Docker, and write a full RAM budget including bots.ac, PG17 and sandboxes. b) Rely on the per-app caps.
15. **audiovau.lt (public exposure).** a) ★ Retire it and remove it from the rules' public list. b) Migrate it into CapRover later.
16. **External monitor timing.** a) ★ Set up the free external monitor before CapRover install. b) Keep it at step 11.
17. **1Password service account (security).** a) ★ Confirm it is read-only on one vault and set a 90-day rotation. b) Leave as is.
18. **GHCR token (security).** a) ★ Create a bot GitHub account for read-only pulls. b) Use `droxey` with `read:packages` and a 90-day expiry.
19. **Update cadence.** a) ★ A monthly calendar slot for CapRover and Docker updates with a backup first. b) Only when needed.
20. **Second backup copy host (data loss).** a) ★ One RackNerd server (nyc). b) The LuxVPS (de) server. c) None.

## Top fixes, in order

1. SSH port in the rules fixed to match the live port (C1, decided). 2. Keep `*.bots.ac`, add per-app and apex records, CapRover catch-all 404 (C3, decided). 3. CI access via auto-refreshed GitHub Actions IP ranges before Task 5 Step 4 (F3, decided). 4. Bot user and sudo before Docker (F11, open). 5. Restic design plus bots.ac DB; drop R2; always keep `~/backups/grokbot` (F7, decided). 6. External monitor and swap before CapRover (F14, F6, open). 7. Postgres on localhost plus tailnet IP after fresh dump (F12, decided). 8. UTC schedule gaps before 2026-11-01 (F8, open). Questions 7-20 remain open.
