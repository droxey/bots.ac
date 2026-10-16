# bots.ac Implementation Plan

> **For the implementing agent:** work task by task. Each task is one PR-sized change: write the failing test, run it and watch it fail, implement, run it and watch it pass, then commit. Use local git plus `gh` only. Every UI task must also pass the mobile and UX checks listed under it.

**Goal:** Turn the Guilds.run fork (`droxey/bots.ac@ca45b04`) into a voice-first, Gather-style pixel world where animated bots work in rooms. It must work on phone and desktop, and it deploys to CapRover at bots.ac.

**Architecture:** see `DESIGN.md`. New code is additive. Server code goes under `apps/orchestrator/src/{world,actionlog,wall,voice,share,costs,mailroom,outreach,memsync,auth}`. Client code goes under `apps/web/src/features/{world,voice,wall,actionlog,share}`. Shared types go in `packages/world-protocol`. Every core file that gets touched is listed in `UPSTREAM.md`.

**Tech stack:**
- Existing: Node 24, Fastify, Postgres 17 (drizzle migrations), React 19, Vite, TanStack, Tailwind v4.
- Added: Phaser 3 (pixelArt), Tiled `.tmj`, easystar.js, `ws`, ElevenLabs (TTS/STT/voice clone), Twilio Voice, `@simplewebauthn`, Web Push (VAPID), Playwright, BrowserStack, axe-core.

**Effort key:** S ≤ ½ day, M ≈ 1 day, L ≈ 2–3 days.

**Standard checks per UI task ("M+UX"):**
- 44 pt targets
- Works at 375×667 portrait and 844×390 landscape
- Keyboard and list-view parity
- axe has no serious violations
- Reduced motion keeps avatars animated
- Captions on any audio
- BrowserStack smoke on iPhone (Safari) and a Pixel (Chrome)

---

## Phase 0: Foundations and deploy (L, about 1 week)

### Task 0.1: Repo hygiene (S)
**Files:**
- Create `UPSTREAM.md`
- Create `docs/bots-ac/README.md`
- Modify `.gitignore` (add `assets/private/`)

**Accept:**
- `scripts/check.sh` passes unchanged.
- The ignore rule is verified with `git check-ignore assets/private/x.png`.

### Task 0.2: CapRover deploy spike (M)
**Files:**
- Create `deploy/caprover/captain-definition`
- Create `deploy/caprover/opensandbox.override.yml` (host-path STATE_ROOT; Docker access only through a socket proxy that allows container create/start/exec/stop/remove, never the raw `docker.sock`; pre-mortem Q3)
- Create `deploy/caprover/README.md`
- Create `deploy/op-inject.sh` (1Password → `caprover api` env update)

**Steps:**
1. Postgres one-click app.
2. OpenSandbox app with the override.
3. Orchestrator app.
4. Run `/health`.
5. Start one agent run that uses the browser.

**Accept:**
- An agent run with a browser works on the server.
- The orchestrator port isn't public until 0.3 lands, so it sits behind CapRover basic auth meanwhile.
- No secrets in git (`gitleaks` clean).
- OpenSandbox can't reach images/volumes/networks/system endpoints through the proxy (a test call returns 403).

### Task 0.3: Passkey sign-in as an `IdentityProvider` (L)
**Files:**
- Create `apps/orchestrator/src/auth/passkey.ts`
- Create `auth/routes.ts`
- Create migration `drizzle/01xx_passkeys.sql`
- Create `apps/web/src/features/auth/SignIn.tsx` (the `signInPage` slot)
- Modify `src/server.ts` (pass extension) and `config` (an `AUTH_MODE=passkey` value)
- Test `test/unit/auth/passkey.test.ts`

**Accept:**
- Registration and login work on desktop Chrome, iOS Safari (iCloud Keychain) and Android (Google Password Manager).
- A recovery code is shown once.
- `compat-cookie` is refused when `NODE_ENV=production`.
- M+UX.

### Task 0.4: Private asset source (M)
**Files:**
- Create `apps/orchestrator/src/world/assets.ts` (an `AssetSource` with R2 and local-volume implementations, plus the `/assets/private/*` route, gated by auth or a share token)
- Create `deploy/assets-sync.sh`
- Create `apps/web/public/assets/cc0/` (Ninja Adventure placeholders)
- Test `test/unit/world/assets.test.ts`

**Accept:**
- Unauthenticated requests get 401.
- `Cache-Control: private`.
- The app runs with the CC0 placeholders when there's no bucket.

### Task 0.5: Nightly backup and export (M)
**Files:**
- Create `deploy/backup/backup.sh` (pg_dump plus attachments tar, age-encrypted, pushed to R2 with 30/12 retention)
- Create a CapRover cron app or host cron doc
- Create `apps/orchestrator/src/export/` (zip export as JSON plus Markdown)

**Accept:**
- A restore drill into a scratch Postgres succeeds and is documented.
- The export opens in a fresh Obsidian vault.
- (Pre-mortem fast-follow) A monthly automated restore drill into scratch Postgres runs as a routine, and a failure is a red `routine_run` entry plus a push.
- (Pre-mortem fast-follow) An external uptime check (outside the CapRover host) pings `/health` every 5 min and alerts Dani by push/email after 2 failures.

---

## Phase 1: MVP, one map, live bots, talk, wall, log, share (about 3 weeks)

The smallest slice that proves the concept: the Pixel HQ map with the Lobby, one team room and one meeting room. 3–5 always-animated bots driven by real runs. Tap-to-talk voice with captions. A planning wall. An action log. A view-only or talk share link. All of it on phones.

### Task 1.1: World protocol package (S)
**Files:**
- Create `packages/world-protocol/src/{events.ts,state.ts}`
- Test `packages/world-protocol/src/events.test.ts`

Content: typed events `botMoved`, `botAnim`, `botStatus`, `bubble`, `emote`, `cardMoved`, `logAppended`, and a snapshot type.

**Accept:** a round-trip encode/decode test passes.

### Task 1.2: Tiled map and avatar sheets (M)
**Files:**
- Create `maps/pixel-hq.tmj`
- Create `maps/README.md` (layer and zone conventions: `room`, `slug`, `desk`, `object`)
- Create `apps/web/public/assets/cc0/avatars/*.png`
- Create `maps/avatars.json` (`idle|walk|talk-<dir>` frame map per sheet)

**Accept:**
- A lint script validates the zones and that every avatar has all 12 animations.
- LimeZu versions load from the private source when present.
- (Fleet audit T3) `maps/ownership.yaml` maps every room to its owning bots and scope; the lint fails unless each bot has exactly one home room (guest memberships in other rooms are allowed and listed separately), or if two rooms claim the same scope. Cards route to the home room. Cards route by room, so there is no router hop. A fixture with one overlap, or a bot with two home rooms, must fail; a bot with one home room plus guest rooms passes.

### Task 1.3: World state service and tick (L)
**Files:**
- Create `apps/orchestrator/src/world/{store.ts,tick.ts,pathing.ts}` (easystar on the collision layer)
- Test `test/unit/world/tick.test.ts`

**Accept:** deterministic tests pass:
- "walkTo desk" reaches the target in N ticks and avoids walls.
- Animation is `walk` while moving and `idle` after.
- There is never a null animation.

### Task 1.4: Run events → world behaviours (M)
**Files:**
- Create `apps/orchestrator/src/world/director.ts` (subscribes to core's generation/tool events: tool call → walk to the tool object, status icon; handoff/@mention → walk to the receiver and bubble; error → "!" emote; done → confetti)
- Modify the core event bus hook (one line, listed in `UPSTREAM.md`)
- Test `test/unit/world/director.test.ts`

**Accept:** replaying a recorded fixture run produces the expected event sequence.

### Task 1.5: World WebSocket (M)
**Files:**
- Create `apps/orchestrator/src/world/ws.ts` (snapshot then deltas, auth or share token, room-scoped)
- Test `test/integration/world-ws.test.ts`

**Accept:**
- A client gets a snapshot in under 200 ms locally.
- A guest token only receives its room.

### Task 1.6: Phaser world view (L)
**Files:**
- Create `apps/web/src/features/world/{WorldCanvas.tsx,scene/HQScene.ts,anim.ts,camera.ts}`
- Lazy route `/world`

The view interpolates server positions, uses integer zoom, pinch and drag pan, culls offscreen bots, and pauses when the tab is hidden.

**Accept:**
- 60 fps with 12 bots on a Pixel 6a and an iPhone 12 (BrowserStack performance trace attached to the PR).
- World chunk under 300 KB gzipped.
- M+UX.

### Task 1.7: Reduced motion and list view (M)
**Files:**
- Create `apps/web/src/features/world/{ListView.tsx,motion.ts}`
- Test `motion.test.ts`

Reduced motion means half-speed idle, fade-teleport and no particles.

**Accept:**
- Every map action exists in the list view (select bot, talk, open wall, open log, teleport).
- Under `prefers-reduced-motion`, the animation is still playing.
- Keyboard-only walkthrough passes.
- M+UX.

### Task 1.8: Bottom sheet / side panel shell, roster, teleport picker (M)
**Files:**
- Create `apps/web/src/features/world/{Panel.tsx,Roster.tsx,QuickSwitch.tsx}` (⌘K plus roster chips)

**Accept:**
- Portrait uses the bottom sheet; landscape and desktop use the side panel.
- Teleport to any bot in at most 2 taps.
- Safe areas are respected on an iPhone with a notch.
- M+UX.

### Task 1.9: Voice adapters (M)
**Files:**
- Create `apps/orchestrator/src/voice/{provider.ts,elevenlabs.ts,fake.ts,routes.ts}` (STT token minting and a TTS stream proxy; keys stay on the server)
- Test `test/unit/voice/*.test.ts`

**Accept:**
- The fake provider passes the contract tests.
- With ElevenLabs, the first audio byte arrives within 800 ms of the text start (logged).
- (Pre-mortem Q1, Dani 2026-10-16) Every TTS/STT call writes a minimal `cost_events` row (credits, bot, run). A hard cap, set by env and later as a layered setting: 1,000,000 ElevenLabs credits/month, warn at 80%, pause voice at 100%, plus a 20,000-credit limit per single call. Accept: a simulated spend at 100% blocks new voice calls with a clear message; a call over 20k credits is cut off and logged. 1.5.7 and 2.5 extend this table rather than create it.

### Task 1.9b: Fast main loop + message inbox (merge/interrupt) (M)
**Files:**
- Create `apps/orchestrator/src/subagents/inbox.ts`
- Modify the core group-dispatcher hook (record it in `UPSTREAM.md`)
- Test `test/unit/subagents/inbox.test.ts`

**Accept:**
- Five messages sent within 2 s produce one reply.
- The p50 reply time stays under 2 s while a long job runs.
- Ordering (fleet audit T2): ships before 1.10, so tap-to-talk is built on the non-blocking loop.

### Task 1.10: Tap-to-talk client (L)
**Files:**
- Create `apps/web/src/features/voice/{useMic.ts,AudioUnlock.ts,TalkButton.tsx,Captions.tsx}`

Hold to talk or tap to toggle. Partial captions. The final text posts to core chat with `modality=voice`. TTS plays with the bubble and caption bar, and the avatar shows `talk` while audio plays. Barge-in stops TTS.

**Accept:**
- Works on iOS Safari after the first gesture.
- Mic denial falls back to text with a clear message.
- Captions can't be disabled.
- The transcript is visible in chat.
- M+UX on iPhone 12/15 and Pixel.

### Task 1.11: Per-bot voice setting with layering (S)
**Files:**
- Create `apps/orchestrator/src/settings/layered.ts` (global < feature < bot < task resolver)
- Modify the agent settings UI (adds a voice picker and "inherited from …" labels)
- Test `layered.test.ts`

Scope (fleet audit T4): the same resolver also holds standing rules and output style (e.g. the ADHD-friendly output block, scrape pause), stored once at the global layer, plus a bot `status: active|archived` field; archived bots don't load into the world or the scheduler.

**Accept:** resolver tests cover all four layers; the UI shows the source; a global rule edited once appears in every bot's resolved prompt with no per-bot edits; an archived bot is absent from roster and scheduler and can be restored.

### Task 1.12: Action log projection (M)
**Files:**
- Create migration `action_log`
- Create `apps/orchestrator/src/actionlog/{project.ts,routes.ts}`
- Create `apps/web/src/features/actionlog/LogPanel.tsx`

Entries: tool call, file change, message, handoff, card move, cost, and routine run (fleet audit T5: start, finish, failure, skipped, duration). Each links to `?run=&call=`.

**Accept:**
- A fixture run produces the expected rows.
- A failing routine produces a red `routine_run` entry and the routine shows its last 10 runs; no routine fails silently.
- The panel opens from a bot tap.
- Filters work on a phone.
- M+UX.

### Task 1.13: Planning wall (L)
**Files:**
- Create migration `cards`
- Create `apps/orchestrator/src/wall/{service.ts,routes.ts,tools.ts}` (a native tool `wall.move_card` for bots)
- Create `apps/web/src/features/wall/{Wall.tsx,Card.tsx}`

Columns Backlog/Doing/Review/Done with phase swimlanes. Pointer and touch drag (long-press), a "Move to…" menu, and undo. When a bot moves a card, the bot walks to the wall object first.

Fleet audit T3 (A5 = b, confirmed by Dani 2026-10-15): bots hand off through the wall or through direct bot-to-bot messages (DMs). Every DM is written to the action log. A DM marked as a handoff (`handoff: true`, or sent via the `handoff` tool) auto-creates a card linked to the DM; ordinary chat DMs don't (A11). A card-based handoff is a card with owner, status, next step and date, routed to the owner's home room via `ownership.yaml`. Tasks for an offline device (e.g. droxeybook) wait as cards in `Waiting on device` until the device's next heartbeat.

**Accept:**
- Drag works on iOS and Android.
- The single-pointer alternative works (WCAG 2.5.7).
- Cards link to the log.
- A wall handoff between two bots creates exactly one card and one log entry.
- Every bot-to-bot DM produces an action-log entry; a DM flagged as a handoff creates exactly one linked card; an unflagged DM creates none.
- A `Waiting on device` card moves to Doing within 1 min of the device reconnecting.
- M+UX.

### Task 1.14: Share links (L)
**Files:**
- Create migration `share_links`
- Create `apps/orchestrator/src/share/{service.ts,routes.ts,guestPolicy.ts,og.ts}`
- Create `apps/web/src/features/share/{ShareButton.tsx,GuestGate.tsx}`

Options: view-only or can-talk (can-talk stays disabled until the 1.9 voice cap is live; pre-mortem Q1), expiry, revoke, passcode, rate limit, tool allowlist (empty by default), audit log. `/r/<slug>` routing.

**Accept:**
- Web Share opens on phones.
- An expired or revoked link gets 410.
- (Pre-mortem Q4) Revoking or expiring a link closes its live WebSocket connections at once, and open guest sockets re-check their token every 60 s. Accept: an integration test revokes a link and the guest socket closes within 1 s.
- A guest can't reach other rooms, memory or secrets (integration tests).
- The OG image renders.
- M+UX.

### Task 1.15: MVP end-to-end and device pass (M)
**Files:**
- Create `e2e/mvp.spec.ts` (Playwright)
- Create `e2e/browserstack.yml`

**Accept:** on 5 devices, a scripted demo passes: open the room, tap a bot, speak, see captions and a reply, watch the bot walk to the wall and move a card, open the log, share the link and open it as a guest. Results are attached.

## Phase 1.5: Subagent sandboxes (about 1.5 weeks, right after the MVP)
- **1.5.1 (S).** `subagents/provider.ts` interface (spawn, exec/respond, steer, stop, delete, cost events) plus a fake.
- **1.5.2 (M).** Agent 37 adapter `subagents/agent37.ts`: instances, exec, `/v1/responses` SSE, stop/delete, and task-scoped secrets. The key comes from 1Password and is set as a CapRover env var. Contract tests use recorded responses.
- **1.5.3 (M).** OpenSandbox fallback adapter on CapRover, used when no Agent 37 key is set. A layered setting `untrustedWorkProvider` (`local` | `agent37`) routes untrusted jobs (guest, student, outreach, unknown repos) to Agent 37 when set (pre-mortem Q3 option c).
- **1.5.4 (M).** `spawn_subagent` tool, limits (3 per bot, 10 overall, layered), a FIFO queue, and idle-TTL sleep or delete. Accept: a 4th spawn queues, and a reaper test finds no instance left past its TTL.
  - **Durable job queue (fleet audit T3).** Migration `jobs` in Postgres (id, bot, goal, state queued|running|done|failed|dead, attempts, next_run_at, lease_until, card_id). Workers claim with `SELECT … FOR UPDATE SKIP LOCKED`; expired leases return to queued; retries back off exponentially (max 3), then dead-letter with a wall card. Accept: a job running when the orchestrator is killed resumes after boot; a job failing 3 times lands in dead-letter with a card; FIFO order holds per bot.
  - **Health monitor and auto-throttle (fleet audit T1).** `subagents/health.ts` samples host load and free memory every 10 s. If 1-min load > 2× cores or free RAM < 15%, new spawns are denied and stay queued (helper waits at the desk, status 'host busy'); running jobs continue; spawns resume after 60 s below threshold. Thresholds are layered settings. Accept: with a faked over-threshold sample, a spawn queues with a log entry saying why; after recovery it starts with no manual action; the talk loop still replies under 2 s.
- **1.5.5 (M).** Steer and cancel. Accept: "stop" cancels within 3 s and partial results are logged.
- **1.5.6 (M).** Helper sprites: walk out, work, return, hand over, fade out; tap for steps, screen and cancel. Accept: M+UX, and 60 fps with 12 bots plus 9 helpers.
- **1.5.7 (S).** Nested action log and costs attributed to the parent bot. Accept: spawns are refused at 100% of the cap. The `cost_events` migration and per-bot meter move here from 2.5 (fleet audit T5); 2.5 adds TTS/STT/telephony meters, banners and UI. Accept also: a sandbox spend appears in `cost_events` under the parent bot.

## Phase 2: Meetings, blockers, stand-up, costs (about 2 weeks)
- **2.1 Speaking queue and meeting room (L).** `voice/queue.ts`, queue chips, Dani priority, barge-in. Accept: 4 bots in a meeting, one voice at a time, captions throughout.
- **2.2 Web Push and PWA (M).** Manifest, service worker, VAPID, iOS install guide. Accept: a push arrives on an installed iOS PWA and on Android.
- **2.3 Blockers (M).** The bot walks to Dani, a chime plays, a push goes out, and an inline answer card appears. Accept: the answer resumes the run.
- **2.4 Daily stand-up at 10:00 ET (M).** Seeded routine, clock tower gathering, a brief of 40 words or fewer per bot, a stand-up card when she's offline.
- **2.5 Cost meters and cap (L).** Extends the `cost_events` table created in 1.5.7. TTS, STT, telephony and sandbox meters plus core usage. 80% warning, 100% pause in the generation gate. Accept: a simulated spend pauses every bot.
- **2.6 Money-only approvals (M).** `spendsMoney` tool flag, approval card plus push. Accept: no other action asks for approval.
- **2.7 Return summary card and overnight replay at 30x (M).**

## Phase 3: Repo rooms and bot computers (about 2–3 weeks)
- **3.1 Repo room template and live-work-first camera (M).**
- **3.2 Smart objects (L).** CI machine (GitHub checks), PR packages, deploy pad (CapRover API), alerts as cracks, Sentry fires, commit mural, health ambience. Each is tappable for its detail.
- **3.3 Bot computer screen (L).** Embed core noVNC live, with "Take over" (touch input on a phone, keyboard toggle). Fleet audit T6: all bot computers share one persistent browser-profile volume (`bots-browser-profile`, A9 = b): each sandbox starts from a copy and writes new logins back when it ends, so logins survive redeploys and are shared across bots. Exception (pre-mortem Q2, Dani 2026-10-16): guest, student and outreach sandboxes get an empty profile, never read the shared volume, and never write back. Accept: log in as bot A, redeploy, bot B is still signed in; two bots running at once don't corrupt the profile; a guest/student/outreach sandbox starts with no cookies and its logins aren't written back; volumes are in the 0.5 nightly backup.
- **3.4 Session recording and replay synced to the action log (L).** rrweb-style or VNC frame capture, scrubbed by log timestamps.
- **3.5 Follow / shadow mode (S). Meaningful emotes (S). Auto status (S).**

## Phase 4: Memory library, search, mailroom (about 2–3 weeks)
- **4.1 `MemoryStore` adapter (L).** Core memory, mem0 and Obsidian Markdown (vault path config), dual-write and swap, last-write-wins with history. Fast-follow (pre-mortem): an outbox retries a failed write to either store, and a nightly diff flags mem0/Obsidian divergence as a wall card.
- **4.2 Library room UI (M).** Browse, edit, and forget with 30-day undo.
- **4.3 Unified search (L).** Postgres FTS over chats, log, cards and memory. Voice query. Results carry a map location and teleport there.
- **4.4 Mailroom channels (L, one PR per channel).** Slack, Gmail, Telegram, then SMS/WhatsApp (Twilio, once 10DLC is done), then inbound phone (Twilio or ElevenLabs Agents).

## Phase 5: Seminar features and team structure (about 2 weeks)
- **5.1 Presenter view (M).** Large captions, follow camera, hidden secrets.
- **5.2 Reasoning and tool-call overlay toggle (S).**
- **5.3 Student sandbox rooms (L).** Share link `role=student`, separate guild and machine, own cost cap.
- **5.4 Replay-as-lesson (M).** Annotated replay with chapter markers.
- **5.5 Drag a bot into a room to join the team (M), and the clock tower for routines (S).**

## Phase 6: Outreach booth, Pine-style, acting for Dani (about 3 weeks)
- **6.1 Outreach task scope and policy engine (L).** `outreach/policy.ts`: contact allowlist, per-task scope, `task_id` on every outbound, emergency and premium denylist (libphonenumber), do-not-contact list, rate limits, audit. Tests cover every block.
- **6.2 Persona profile (M).** Library entry: tone, phrasing excerpts (approved by Dani), negotiation limits, sharable facts. Prompt builder. Tests check that the script never claims to be Dani and that limits are enforced on the server.
- **6.3 `TelephonyProvider` (L).** Twilio Media Streams plus bot STT/TTS, and ElevenLabs Agents outbound. Fixed opener template with no "AI" wording and a recording notice on every call. Tests check that the bot never denies being automated when asked directly. A fake provider for tests.
- **6.4 Live listen-in and transcript, plus "Join / Take over" (L).** Twilio Voice JS on desktop and mobile, with a phone-bridge fallback. Accept: takeover in 2 taps or fewer on an iPhone.
- **6.5 Hold detection and a push when a human answers; verification always handed to Dani with a tap and a push (M).**
- **6.5b Receive-first SMS (S).** A Twilio number receives verification codes and routes each to the task that asked; no A2P registration needed. Outbound SMS stays off until A2P is done. WhatsApp outbound is out of scope.
- **6.5c Audio retention (S).** Transcripts are kept. Audio is deleted unless confidence is below 0.85, there are gaps, or the transcript is incomplete; then the card is flagged "check audio". Tests cover both paths.
- **6.6 Money gate in the call flow (S).** Reuses 2.6.
- **6.7 Outcome card on the wall, plus recording and transcript in the action log (S).**
- **6.8 Optional voice clone of Dani (M).** In-app sample recording, ElevenLabs clone, revocable. Stock voice now; the clone ships later, opt-in.
- **6.9 Email and chat outreach under the same scope (M).**

## Phase 7: Migration, polish, upstream (ongoing)
- **7.1 Opt-in Grok Bot fleet importer (L).** Per bot and per item, tagged, can be rolled back. Fleet audit T4: dedups standing rules repeated across profiles into one global rule (listing which bots had it), imports parked/hidden bots as `archived`, flags overlapping skills (essence-*, lesson-plan, Cloudflare handoff) for merge. Accept: a fixture where 3 profiles share a rule yields 1 global rule and 0 per-bot copies; a parked bot imports archived.
- **7.2 Swap in LimeZu art and credits page (S).**
- **7.3 Monthly upstream merge from CigaleAI/guilds (S, recurring).** Uses `UPSTREAM.md`.
- **7.4 Full WCAG 2.2 AA audit plus a BrowserStack regression matrix (M).**

## Assumptions
Fleet rule: every assumption is written here with a confidence score and later asked to Dani as multiple choice with a recommended answer. Added 2026-10-15 while folding in `FLEET-AUDIT.md`.
- **A1. Throttle thresholds:** deny new spawns at 1-min load > 2× cores or free RAM < 15%; resume after 60 s below. Options: a) this [recommended], b) load > 1× cores, c) RAM only. Confidence: 60%. **Confirmed by Dani 2026-10-15 (a).**
- **A2. Job queue in existing Postgres 17** (SKIP LOCKED), no Redis. Options: a) plain Postgres [recommended], b) pg-boss, c) Redis/BullMQ. Confidence: 85%. **Confirmed by Dani 2026-10-15 (a).**
- **A3. Retries:** max 3, exponential backoff, then dead-letter card. Options: a) 3 [recommended], b) 5, c) no auto-retry. Confidence: 75%. **Confirmed by Dani 2026-10-15 (a).**
- **A4. Rooms per bot:** exactly one home room plus any number of guest rooms; lint enforces one home room. **Confirmed by Dani 2026-10-15 (b, changed from recommended a).**
- **A5. Handoff channels:** wall plus direct bot-to-bot messages; every DM logs to the action log. **Confirmed by Dani 2026-10-15 (b, changed from recommended a).**
- **A6. 1.9b moves before 1.10** inside Phase 1, no schedule change. Options: a) yes [recommended], b) keep after 1.15. Confidence: 90%. **Confirmed by Dani 2026-10-15 (a).**
- **A7. `cost_events` ships in 1.5.7**, 2.5 extends it. Options: a) yes [recommended], b) ship all of 2.5 before Phase 1.5. Confidence: 85%. Confirmed by Dani 2026-10-15 (a).
- **A8. Archived bots** stay in the DB hidden and restorable, never auto-deleted. Options: a) archive [recommended], b) export then delete. Confidence: 90%. Confirmed by Dani 2026-10-15 (a).
- **A9. Persistent browser profile** = one shared login volume (`bots-browser-profile`) used by all bot computers, in nightly backups. Options: a) one volume per bot computer [was recommended], b) one shared volume, c) none. Confidence: 80%. Confirmed by Dani 2026-10-15 (b, changed from recommended a). Note: Chromium locks a profile while it is open, so each sandbox starts from a copy of the shared profile and new logins are written back when it ends (newest login wins).
- **A10. Offline-device cards** wait with no expiry. Options: a) no expiry [recommended], b) expire after 7 days, c) after 24 h. Confidence: 60%. Confirmed by Dani 2026-10-15 (a).
- **A11. DM handoffs and cards:** a DM explicitly flagged as a handoff auto-creates a linked wall card; ordinary DMs only log. Options: a) flagged DMs create a card [recommended], b) every DM creates a card, c) DMs never create cards (log only). Confidence: 65%. Confirmed by Dani 2026-10-15 (a).

### Pre-mortem answers (2026-10-16, from `ecc-adapt/tests/premortem-bots-ac-2026-10-16.md`)
- **A12. MVP voice spend cap:** per-call `cost_events` plus a hard cap in 1.9 (1,000,000 credits/month, warn 80%, pause 100%, 20,000 credits/call); can-talk links off until then. **Confirmed by Dani 2026-10-16 (a).**
- **A13. Shared browser profile:** guest, student and outreach sandboxes get an empty profile. **Confirmed by Dani 2026-10-16 (a).**
- **A14. Docker access:** socket proxy limited to container ops by default; the `untrustedWorkProvider=agent37` setting is available. **Confirmed by Dani 2026-10-16 (a + c as a setting).**
- **A15. Share revoke:** close live sockets on revoke/expiry, re-check every 60 s. **Confirmed by Dani 2026-10-16 (a).**
- **A16. Host:** bots.ac runs on a separate host (Servarica Montreal CapRover), not the Grok Bot box. **Confirmed by Dani 2026-10-16 (a).**
- **A17. ElevenLabs grant credits:** the Muse & Machine account (Grant Tier 2) shows about 32.9M credits, and it's unknown whether grant credits renew monthly or expire. Confidence: 50%. Options: a) treat them as a one-time balance with an expiry, keep the 1M/month cap and check the expiry date in the ElevenLabs billing page [recommended], b) assume monthly renewal and raise the cap, c) ask ElevenLabs support first. Confirmed by Dani 2026-10-16 (a): one-time balance; the ElevenLabs grant (Grant Tier 2, Muse & Machine) ends 2027-08-02, so keep the 1M/month cap and re-plan voice before that date.
- **A18. Outreach login lease:** optionally, an outreach task may get a per-task, Dani-approved login lease for a single site (cookies for one domain copied in, discarded at the end, never written back). Confidence: 55%. Options: a) not now; outreach uses an empty profile and hands off to Dani for logins [recommended], b) add the lease with an approval card per task, c) never. Confirmed by Dani 2026-10-16 (b): outreach may lease one site login per task, with an approval card each time; cookies copied in for that task and discarded after.
- **A19. Backup decryption key:** the age private key for nightly backups lives in 1Password (bots.ac vault) with a printed recovery copy, and is never stored on the CapRover host. Confidence: 60%. Options: a) 1Password plus a printed copy [recommended], b) 1Password only, c) on the host (not recommended: a host compromise exposes the backups). Confirmed by Dani 2026-10-16 (a): backup decryption key in 1Password plus a printed copy, never on the CapRover host.
