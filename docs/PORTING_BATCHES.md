# Android port: review and batch plan

Written 2026-10-10 against PORTING.md, PORTING_ROADMAP.md and ANDROID_PORT_LOG.md (the versions shared that day)
and this iOS checkout at `21a6320` (bighelp 2.3.0 (104)), which is the commit the fork last merged. Everything
marked **verified** was read from this checkout's source; everything else comes from the three documents. When a
line here and the Swift source disagree, the source wins.

Part 1 is where the port stands. Part 2 is the suggested changes to the three documents. Part 3 is the plan: the
remaining work cut into batches that a person or an agent can take cold, each with scope, what to port from, what to
build, how it is accepted and what evidence it must leave behind.

---

## 1. Where the port stands

| Package | State | Evidence so far | Still open |
|---|---|---|---|
| WP1.1–1.3 scaffold, wire, HTTP | Done | JVM unit | — |
| WP1.4 authentication | Done | JVM unit; **live** PKCE and bearer on the gated host (2026-10-05) | token/dashboard/password live rows never run |
| WP1.5 transport | Done | fake socket, MockWebServer; live ping and resume | 30 s network cut with a fresh ticket (unit only) |
| WP1.6 sessions | Done | JVM; live create/resume/capabilities | — |
| WP1.7 projection | Done | golden fixtures from a real host | P3 surfaces not ported: todo snapshots, side tasks, reactions, goal/loop control |
| WP1.8 submit/steer/stop | Done | JVM; live steer during a tool, interrupt | — |
| WP1.9 recovery | Done | JVM; live two-socket resume | the "drop the network 10 s mid-stream" live row |
| WP1.10 plugin context and routes | Done | JVM; live `people/speaking` | principal check present; route table covers MVP rows only |
| WP1.11 prompts | **Partly**: approval and clarify (WP2.8) | fake host | secret, sudo, `vault.*`, `mcp.setup` server requests; legacy event prompts |
| WP1.12 downloader, reply quote | Done (in WP3.1, WP3.2) | fake plugin; JVM | live `MEDIA:` turn on a disposable host |
| WP1.13 CLI driver | Done | live | the full §7 command list was never completed; fine, the app took over |
| WP2.1–2.9 Android shell | Done at fake-host level | JVM flow tests; Compose tests compile only | every Compose test is unrun (no emulator on the box) |
| WP2.10 Phase 2 device exit | **Not formally run** | 2026-10-10: 0.1.8 connects and chats on the Pixel; WP3.4 actions checked | cold relaunch after a kill, airplane mode, Doze, Q2 browser loopback, TalkBack, font scale |
| WP3.1 agent files | Done, one defect | fake plugin | **APK attachments don't work on the device**, cause unknown; hosted `https` pictures not fetched; video playability not checked |
| WP3.2 reply quoting | Done | fake host | Pixel checks for the long-press menu |
| WP3.3 file uploads | Done | fake host | real `file.attach` on a host; picker with real providers |
| WP3.4 session management | Done | fake host; **device** (Eric, 2026-10-08) | — |
| WP3.5 background continuity | Kick-off done, two real bugs fixed | fake soak | the 24 h Pixel soak; false "unconfirmed" notice (cause known, not fixed) |
| WP3.6 Feed / Ideas / Goals | Read-only slice | fake board; **device** empty state and one test post | every mutation; post files; web pictures; unread dots |
| WP3.7 cards | Static slice | fake host | card actions; a real card from a disposable host |
| WP3.8 commands, model, reasoning | Done | fake host | Pixel checks |
| WP3.9 notifications | Not started | — | owner decision D2 |
| Phase 4 Windows | Not started | — | — |

Versions shipped: 0.1.0 (1) → 0.1.9 (10), debug-signed APKs over LAN HTTP. 632 JVM tests, cold suite green on
0.1.9 after the sign-in race fix (`b428c5c`) removed the long-standing flake.

Reading the log as a whole: the protocol core and the chat path are in good shape and were proven against a real
host early, as the roadmap asked. What is thin is **device evidence** (almost every Compose surface is compile-only)
and **decisions** (D1–D10 are still "open" in the documents while the code has quietly chosen most of them). The
batches below put both first.

---

## 2. Suggested changes to the documents

### 2.1 PORTING.md

1. **Close the decisions the code already made, and make the two that get more expensive every day.**
   - **D1 app identity.** `net.ebev.bighelp` has shipped to the Pixel ten times. Changing `applicationId` later
     means a fresh install: the Keystore vault, saved chats and drafts on the phone are lost, and every installed
     copy has to be removed by hand. Decide now, before a second person installs. The decisions table should carry
     the chosen package and display name, not a TODO.
   - **D3 distribution and signing.** The APKs are debug-signed. Android refuses to install a release-signed build
     over a debug-signed one, so the first release key also forces an uninstall. Create the release keystore,
     sign from the next build on, and record the key's SHA-256 in the private notes. Gitea releases as the channel
     is fine; add the `versionCode` rule (monotonic, one per published APK) to the table.
   - **D7 person ID**: the port uses one random ID per install. Record that as the decision for now, with the
     consequence (two Android devices of one person look like two people in group chats; iOS devices of the same
     person look like one).
   - **D8 plain HTTP**: built and lint-flagged on purpose; record as decided.
   - **D9**: "update the plugin" wording only, no installer. **D10**: banner plus "Review unsent drafts", no
     recovery view. Both are built; record them.
2. **Notifications (open question 1) now has facts.** The broker in `services/link` is a BuzzKit integration
   (`services/link/src/buzzkit.ts`, **verified**): subscription rows carry `channel`, `provider` and `platform`,
   and the code filters on `provider === "apns"` and `platform === "ios"`. So Android push is not a host or plugin
   change (the locked rule holds) but it is a **broker change plus a BuzzKit question** (does BuzzKit deliver to
   Android, and through what), and only the upstream maintainer deploys the broker. Independently, the plugin's
   `alerts/listen`, `alerts/ack`, `alerts/stop` routes (`native-live-alerts-v1`, **verified** in
   `DirectHermesNativeContext.swift`) deliver the same sealed alert while the app is open, with no broker. Split
   the question: in-app alerts can be built now (Batch 7a); push waits on the maintainer (Batch 7b).
3. **Wire contract counts.** "47 in the table" is stale: the route switch now builds 49 `Route(...)` entries over
   50 distinct paths (workflows is one parameterized case), plus the four separately coded families. The workflows
   gate is now three features: `native-workflows-v1`, `native-workflows-edit-v1`, `native-workflows-trigger-v1`
   (**verified**). Say "about 50, re-count at each sync" rather than a number.
4. **Upstream sync: add a port-relevant path list** so the daily digest can flag what matters: `Bighelp/DirectHermes/`,
   `Bighelp/Workspace/WorkspaceOperation.swift`, `Bighelp/Chat/ChatModel*.swift`, `Bighelp/Hermes/`,
   `docs/NATIVE_TRANSPORT.md`, `docs/CHAT_INTERACTION_CONTRACT.md`, and in the plugin repo `loopdy_plugin/native_*.py`
   and `spec/`. Everything else can batch.
5. **Add a "features the port does not have" line** that mirrors upstream's current screens, so nobody thinks
   they were forgotten: Scheduled tasks, Kanban (a separate Hermes plugin at `/api/plugins/kanban/*`, **verified**),
   Workflows, Usage, Projects, Agent Studio and the avatar/template catalogs, Fast Mode, session search, App layout.
   Part 3 gives each a batch or a deferral.

### 2.2 PORTING_ROADMAP.md

1. **§2.2 is missing a whole RPC family the app ships: scheduled tasks.** `scheduled_tasks.list / create / update /
   delete / pause / resume / run / delivery_targets`, events `cron.changed`, `scheduled.completed`,
   `scheduled.failed`, and REST `GET /api/cron/blueprints`, `POST /api/cron/blueprints/instantiate`,
   `GET /api/cron/delivery-targets` (**verified**; client `DirectHermesScheduledTasksClient.swift`, screen under
   `Bighelp/Scheduling/`). Add the rows with MVP "—" and point Batch 10 at them.
2. **§2.5 route table**: add `alerts/listen|ack|stop`, `board/accept`, the workflows family with its three gates,
   `workflows/trigger/set`, and note the 192 KiB request cap now also covers the workflows features
   (`smallRequestFeatures`, **verified**). Re-state the count as in 2.1.3.
3. **§2.6 dashboard REST**: add `/api/analytics/usage` and `/api/analytics/models` (Usage page), `/api/cron/*`
   (above), and the `source / sources / exclude_sources` filters on `/api/sessions/search` (**verified** in
   `DirectHermesWorkspaceClient.swift`), which is how upstream now shows "other-app chats".
4. **C4 (conflict retry) disagrees with the shipped port.** The roadmap says the route client never retries on 412
   and callers do; the port's `PluginRouteClient` reloads the context and resends **reads** once inside the
   client and never resends mutations (log, WP3.1). That is the safer rule and it matches AGENTS.md. Change C4's
   impact column to describe it and stop calling it a deviation in the log.
5. **§3.3 mid-turn send**: the port replaced the long-press menu with a visible "More ways to send" button for
   TalkBack reasons. Record it in the roadmap as the Android design, not only in the log.
6. **WP2.6/WP3.2 "long-press › Reply, as on iOS"** needs a check before anyone claims parity. In this checkout the
   bubble menu is `ChatBubbleMenuAction { react, copy, selectText, forkFromHere }`
   (`Chat/ChatMessagePresentation.swift`, **verified**); Reply is drawn by `ChatReplyViews.swift` and may not be a
   menu item. Fork isn't available on native chats (`session.branch` is disabled for Direct), so the port's choice
   to skip it is right. Fix the wording: "Reply, Copy, Select text; React when reactions land (Batch 8); no Fork."
7. **§6.3 persistence layout** has drifted from what shipped: add `noBackupFilesDir/preferences.json`
   (`AppPreferencesFile`), `noBackupFilesDir/person-id`, the transcript store at
   `files/transcripts/<sha256(hostIdentity)>` (one manifest per computer), and the attachment cache at
   `cacheDir/agent-attachments/<sha256(host)>/<key>/`. Keep the table as the authority and make the log point at it.
8. **§7 WP1.11 is not done.** Secret, sudo, `vault.*` and `mcp.setup` server requests and the legacy event prompts
   are unported; WP2.8 covered approval and clarify only. Mark the remainder and give it a batch (Batch 5).
9. **§9 test strategy: write down two rules the log has been following by hand.** (a) RED first: every new class is
   observed failing (compile failure counts) before the code exists, and the log names the mutation that proves a
   test bites. (b) A cold suite with a known flake is **not green**; the flake gets a root cause (WP3.5 found one in
   `SignInController`). Add a third: Compose `androidTest` classes are "compile-only" until an emulator or device
   runs them, and the log must say so (it does).
10. **§10.3 add Q10**: the false "Message delivery is unconfirmed" notice when the app is backgrounded during an
    accepted stream. Root cause is in the log (WP3.5): `suspendConnection` ends the pending submit with
    `Disconnected(outcomeUnknown = true)`, the notice shows at once, and the exact replay then retires the entry
    (`acceptedRecovery`) without clearing the notice. Batch 3 fixes it.
11. **Fold the resolved rows** (C11, D6) into the running text and leave their history to the log. The roadmap
    reads better as "what is true" than as a diff of itself.
12. **Add the Android design authority.** DESIGN.md is iOS. The Material 3 canvas
    (https://claude.ai/artifact/XG4XiVdEK4J8Du5AUKCyWY: chat light/dark, drawer, settings, token sheet, and the
    Appearance screen with the Match-wallpaper option) is the Android look until a `DESIGN_ANDROID.md` exists;
    Batch 6 writes that file from it.

### 2.3 ANDROID_PORT_LOG.md

1. **Put a status table at the top** (the one in Part 1, kept current per batch). A 350-line log with no index is
   where "is X done?" questions go to die.
2. **Dates are out of order** (WP3.4, dated 2026-10-07, sits after WP3.2/3.3 dated 2026-10-08) although the header
   says "newest last". Either reorder or change the rule to "grouped by package".
3. **Redact the LAN address and home path** (`http://10.0.0.23:8088/…`, `/home/edi/public-downloads`). PORTING.md
   asks for no real hosts or addresses in docs even though the fork is private, and a copy of this log will end up
   in a chat transcript sooner or later. `<lan-download-host>` carries the same meaning.
4. **One line per deviation is the rule; the WP3.1 and WP3.6 entries are essays.** Move the durable facts (limits,
   layouts, deviations) into the roadmap's tables and keep the log to what happened and why.
5. **Record the APK-attachment defect as a numbered bug**, not a paragraph inside a success entry, with the
   triage steps (does the tile show; does `resolve` list it; does `fetch` finish; does the Open hand-off fail).
   Batch 4 owns it.

---

## 3. Batch plan

### 3.1 How a batch is written and closed

Every batch below is one PR. The brief an agent gets is the batch section plus these standing rules:

- **Read first:** the roadmap sections named in the batch, then the Swift files named under "Port from". Behavior
  is ported, not structure. Every limit that iOS has, Android keeps unless the batch says otherwise.
- **Fakes behave like the host.** A new fake behavior needs a recorded transcript or a read of the stock source
  (name the file, as the log does). A fake that answers `queued` on an idle submit is a bug.
- **RED first.** Each new test class is observed failing before the code exists; the log names one mutation per
  batch that makes a test fail.
- **Evidence labels** on every result: JVM unit, fake host, compile-only, disposable host, real host, device.
- **Done means:** cold `:app:testDebugUnitTest --rerun-tasks` 0 failures (a flake is a bug, not a note),
  `assembleDebug`, `assembleDebugAndroidTest`, `lintDebug` 0 errors, `git diff --check` clean; version bumped
  (`versionName` and `versionCode`); the log gets its entry (status table row updated, one line per deviation);
  a **device checklist** for Eric is written into the log, short enough to run in ten minutes.
- **Never:** change the host or plugin; send anything the iOS app doesn't send; create a chat, turn, cron job or
  board item without the person's explicit tap (no AI spending by default); put a host path, token or real address
  in a log line, a test or a doc.

Sizes are the roadmap's: S ≤ 3 days, M ≤ 2 weeks, L > 2 weeks, one engineer. Order is by dependency and by what
Eric sees first; batches with no arrow between them can run in parallel on separate branches.

```
B0 decisions ─► B1 device exit ─► B2 delivery fixes ─► B3 agent files ─► B4 boards ─┐
                                                                                       ├─► B6 Material 3
B5 secure prompts (any time after B1)                                                  │
B7a in-app alerts (after B2)          B7b push (blocked on maintainer)                 │
B8 reactions + menu  B9 settings  B10 scheduled tasks  B11 hardening  ◄────────────────┘
B12 cards actions    B13 usage    B14 workflows   B15 kanban   B16 agents   B17 voice
```

### B0. Decisions and document fixes — S (owner, half a day; agent for the edits)

- **Decide:** D1 package and display name (see 2.1.1), D3 release key and `versionCode` rule, D7/D8/D9/D10 as
  recorded above. Write them into PORTING.md's table with the date.
- **Edit:** every item in Part 2. The roadmap's §2.2, §2.5, §2.6, §6.3 tables get the missing rows; C4 is
  reworded; Q10 is added; the log gets its status table and redactions.
- **Acceptance:** PORTING.md has no "open" decision older than a week without a named owner and date; the
  roadmap's tables match `DirectHermesNativeContext.swift` and `WorkspaceOperation.swift` at `21a6320` (an agent
  can diff the route paths and feature strings mechanically).
- **Device checklist:** none.

### B1. Phase 2 device exit and the open Pixel checks — S (mostly Eric's time)

Everything the log lists as "device-only, still open", in one pass, on 0.1.9 or the first release-signed build.

- **Run:** (1) cold relaunch: `adb shell am kill <pkg>` while backgrounded mid-reply, reopen: history and draft
  present, no duplicate turn; (2) airplane mode 2 minutes mid-reply, off: reconnects alone, reply completes once;
  (3) background 10 minutes, foreground: catches up, nothing resent; (4) Doze (`dumpsys deviceidle force-idle`,
  `unforce`), open: reconnects; (5) Q2: browser sign-in with the public https address, leave the browser in front
  5 minutes before finishing, confirm the loopback page and the return; (6) the WP3.2/3.8/menu checks (long-press
  menu with Reply / Copy / Select text; "/" commands; model and reasoning sheet; expensive-model dialog);
  (7) light and dark, font scale 1.0 and 1.3, TalkBack on the chat list, composer, send choices, unconfirmed
  notice, prompt cards; (8) the APK-attachment triage steps (which step fails: tile, resolve, fetch, Open).
- **Record:** each as a **device** line in the log; each failure becomes a numbered bug for B2/B3/B4.
- **Acceptance:** the PORTING.md Phase 2 exit sentence is true on the device and the log says so with a date.
  Q2 is closed either way.

### B2. Delivery certainty: the notice that lies, and two races — M

The chat's core promise is "never resend, never lie about it". Three known cases bend the second half.

- **Port from / read:** `+Submission.swift` `completeSubmissionIfReady`, `+Recovery.swift` `acceptedRecovery`,
  `ChatModel+Submission.swift` (the notice copy), roadmap §3.2 step 16, §3.5; the log's WP3.5 analysis.
- **Fix 1, the false "unconfirmed" notice (Q10).** Rule: the notice is shown only when a catch-up pass has
  finished and the entry is **still** unresolved, or when no pass can run (no connection after the backoff gives
  up). A `Disconnected(outcomeUnknown = true)` on `suspendConnection` marks the entry "awaiting catch-up" and
  shows nothing; the exact replay that retires the entry (`acceptedRecovery`) clears that mark; a pass that ends
  with the entry unresolved shows iOS's copy word for word. The entry itself never changes; only when the notice
  shows does.
- **Fix 2, "unconfirmed although never dispatched"** (log, WP3.3): a prepared `prompt.submit` that fails at the
  admission ping, before any frame was written (`sendStarted == false`), is a definite not-sent. Retire the entry
  and hand text and files back with "wasn't sent" copy. Only `sendStarted == true` may ever read as uncertain.
- **Fix 3, `stop()` tail over a new `start()`** (log, WP3.5 "not done"): a stop that finishes after a later start
  must not mark the pill Offline. Give each start a generation; a stop only writes state for its own generation.
- **Tests (fake host, RED first):** `BackgroundContinuityFlowTest` gains "background mid-reply shows no notice
  and the entry is retired on return"; `SendChoicesFlowTest` gains "a lost receipt after a sent frame still shows
  the notice after the pass" (the behavior that must not regress); `UploadFlowTest` gains "no socket at admission
  hands everything back"; `ConnectionUpkeepFlowTest` gains the stop/start race with a slow stop.
- **Device checklist:** send, press Home while the reply grows, wait a minute, return: reply complete, **no**
  notice. Airplane mode right after Send, before the reply starts, keep it on 2 minutes: notice **does** show
  after reconnect if the host never took it, and the text is in Review unsent drafts.

### B3. Agent files: finish the feature — M

- **Bug: APK attachments** (device-reported). Triage from B1 decides the fix; likely candidates are the MIME the
  provider reports (`application/vnd.android.package-archive`) versus the `MediaMarkers` table, or the
  `ACTION_VIEW` hand-off needing `FLAG_GRANT_READ_URI_PERMISSION` plus the package-installer intent. Whatever it
  is, the fix lands with a fake-resolver test and a device line.
- **Hosted pictures (roadmap §4.6).** Port `LinkPreviewPolicy.loadableURL` as `ProviderUrlPolicy` (host-string
  check, no DNS; rejects IPv6 literals, dotless names, `localhost`, the private suffix list, private/reserved
  IPv4) and a bounded fetch: ephemeral client, no cookies, no cache, 10 s request / 20 s total, ≤ 25 MiB, 2xx,
  `image/*` or `video/*`, redirects re-checked against the same policy. Used for a tool result's `https` picture in
  chat and for a board item's `{url}` picture (replacing WP3.6's "Tap to open" tile).
- **Video playability** (roadmap §4.3 `Validating`): a temp copy in a 0700 folder under `noBackupFilesDir`, probed
  with `MediaMetadataRetriever` (duration or a frame), deleted at once; failure is `Failed`, not a crash.
- **`attachments/board`** (`native-agent-board-files-v1`, 60 s, 16 KiB): `{agentId, itemId, index}` → one
  attachment, then the same chunked fetch. Feeds B4's post files.
- **Tests:** `ProviderUrlPolicyTest` (every rejection class), `HostedPictureFetchTest` (MockWebServer: redirect
  to a private host refused, size cap, wrong MIME), `AttachmentResolverTest` board route, fake-plugin flow for a
  board file.
- **Device checklist:** an agent turn that returns an APK; a picture from a hosted image tool; a short video;
  a Feed post with a file.

### B4. Feed, Ideas and Goals: the mutations — M

Port from `Bighelp/Board/` (`AgentBoardStore` write paths), plugin `loopdy_plugin/native_api.py` (read-only), and
the WP3.6 fake `FakeBoard`, which needs the mutation routes added with the plugin's exact checks.

- **Read/unread:** iOS marks an item seen after 1.5 s on screen; `board/read` (`native-agent-board-feedback-v1`,
  16 KiB). Unread dots on tabs and rows come from `read == false`.
- **Rate, dismiss, undo, goal done:** `board/update` (2 MiB response cap stays). One change in flight per item;
  nothing optimistic (the row changes when the host answers), as WP3.4 did for sessions.
- **Promote** an idea: `board/promote`. **Accept** a peer answer: `board/accept` (`native-agent-board-answers-v1`,
  mutation); hide the control when the feature is absent.
- **"Ask <agent>" and "Create a goal"** open the chat with prefilled text and **do not send**: the person taps
  Send. This is the "no AI spending by default" line; a test asserts no `prompt.submit` on the tap.
- **Post files** through B3's `attachments/board`; **web pictures** through B3's fetch.
- **Tests:** `FakeBoard` mutations (428/412/422/404 in the plugin's order, as the read fake already does);
  `BoardFlowTest` gains one case per mutation plus "Ask sends nothing".
- **Device checklist:** rate and dismiss a Feed post, undo it; mark a goal done; promote an idea and see it on
  Goals; tap Ask and confirm the message box is filled and nothing was sent.

### B5. Secure prompts and the legacy prompt contract — M

The rest of WP1.11.

- **Port from:** `DirectHermesSecurePrompt.swift` (`DirectHermesSecurePromptStore`), `+Prompts.swift`,
  roadmap §2.4 and the prompt-response table in §2.2.
- **Build:** `SecurePromptStore` for server requests `secret`, `sudo`, `vault.code`, `vault.save_login`,
  `vault.unlock_prompt`, `mcp.setup`: answer `{value}` on the same request ID, empty string declines,
  `vault.save_login` value is JSON `{identifier, password}`; the entry field is never logged, persisted or put in
  a draft; `request.cancel` removes the card; a new connection retires everything. Legacy path when
  `open_requests` is absent: `secret.respond {request_id, value}`, `sudo.respond {request_id, password}`,
  `approval.respond`, `clarify.respond` (answer `""` on dismissal), sent once, never replayed.
- **UI:** one card style with a password field (`KeyboardType.Password`, no autocorrect, no suggestions,
  `FLAG_SECURE` on the chat window while a secure card is open), copy from iOS. Resolve C9 the way the roadmap
  says: follow the secure prompt store, not the `+Events` status string.
- **Tests:** `SecurePromptStoreTest` (each request kind, decline, cancel, retire), `LegacyPromptFlowTest` (fake
  host without `open_requests`), screenshot-free Compose test for the password field flags.
- **Device checklist:** `Scripts/HostProbePlugin`'s secret tool on a disposable host; the field shows dots; the
  value never appears in `logcat`.

### B6. Material 3 and Material You — M

From the design canvas (chat light/dark, drawer, settings, token sheet, Appearance with Match wallpaper). This
batch gives the app one look; it doesn't add features.

- **Theme:** `Color.kt`/`Theme.kt` from the token sheet (Ember cream and lavender as the default roles; dark
  "After dark" surfaces; coral as tertiary), dynamic color behind the Appearance choice, light/dark/automatic.
  Outgoing bubbles use `primary` in light and `primaryContainer` in dark, both as drawn on the canvas.
- **Appearance screen:** Colors (bighelp / Match wallpaper, the latter hidden below Android 12), the 12 bubble
  colors (`BighelpBubbleColor`, **verified** hexes in `BighelpAppearanceChoices.swift`), Light page Cream/Paper,
  Dark page Graphite/Black, Automatic/Light/Dark. Keys follow iOS's `loopdy.*` names where one exists (the
  cron-filter precedent).
- **Navigation:** the bottom `NavigationBar` (Chat, Feed, Ideas, Goals, Files when it exists) and a
  `ModalNavigationDrawer` for ☰ in the canvas's order: computer switcher, New chat / Group, Agents, Projects,
  Kanban, Workflows, Scheduled tasks, Usage, Settings, Recent chats, More. Rows for places the port doesn't have
  are **absent**, not disabled. Tablets: `WindowSizeClass`, chat lane capped as on iPad, one menu.
- **Icon:** adaptive app icon plus a themed (monochrome) icon from the Ember mark; lint's "no app icon" goes away.
- **Write `docs/DESIGN_ANDROID.md`** from the canvas's token sheet and the iOS → Android table, as the authority.
- **Tests:** theme unit tests for the role mapping; Compose screenshot tests are compile-only here, so the device
  checklist carries the look.
- **Device checklist:** both color choices in light and dark; a wallpaper change updates the app; font scale 1.3;
  the drawer with TalkBack.

### B7. Notifications — 7a S, 7b L (blocked)

- **7a, in-app alerts (now).** `alerts/listen` (long-lived read, re-opened after it returns or the socket moves),
  `alerts/ack`, `alerts/stop` (`native-live-alerts-v1`). Port `BighelpLiveAlerts.swift` for the alert shape and
  `BighelpSealedAlerts.swift` for opening the sealed title, text and picture, and `BighelpNotificationGrouping`
  for the rules: one channel per computer, one group per **agent**, a newer reply replaces that chat's earlier
  ones, questions and approvals stay, helper results are silent, opening a chat clears its alerts. Posted only
  while the app process is alive (the socket lives only in the foreground, by design); the chat on screen never
  gets a banner. Add the `POST_NOTIFICATIONS` runtime ask on first use, not at launch.
- **7b, push (blocked).** What the app would need once the broker can address Android: registration against
  `/v1/notifications/bootstrap`, `/installations/current`, `/host-grants` with a `platform: "android"` row and
  an FCM (or other provider) token, the sealed-alert decryption from 7a in a receiver, and the grouping above.
  The decision belongs to the owner and the upstream maintainer: whether BuzzKit delivers to Android, and
  whether a broker change is welcome upstream. Until then, the app says nothing about push.
- **Tests (7a):** fake plugin `alerts/*` with a sealed fixture; grouping unit tests ported from
  `BighelpNotificationGroupingTests` if present; a flow test that an alert for the open chat posts nothing.
- **Device checklist (7a):** a reply in another chat posts a notification; a second reply replaces it; tapping
  opens that chat and clears it.

### B8. Reactions and the message menu — S

- **Port from:** `ChatModel+Reactions.swift`, `DirectHermesReactionHistory.swift`,
  `NativeMessageReactionMapping.swift`, `ChatMessagePresentation.swift` (**verified** menu order:
  React, Copy, Select text, Fork).
- **Build:** `config.get display.message_reactions` decides availability; `message.react` mutation (shape from
  `DirectHermesSessionActions.swift`); `message.reaction` event and reaction hydration on activation (the WP1.7
  "not ported" item); a reaction bar from the long-press menu. No Fork (not available on native chats).
- **Tests:** projection golden for `message.reaction`; `ReactionsFlowTest` (toggle, host-driven update, hidden
  when the config says off).
- **Device checklist:** react to a message; see the reaction after reopening the chat.

### B9. One Settings screen — S

AGENTS.md: one ☰, one Settings, each row opens one page. The port has settings scattered (cron switch on the
list, name entry nowhere yet). Build the Settings page from the canvas: **Your name** (the speaker note's name,
≤ 40 characters, "You" never sent), **Appearance** (B6), **Chat** (default mid-turn choice Steer / Queued /
Interrupt and send; show scheduled runs), **Notifications** (B7a), **Computers** (the existing list), **About**
(version, licenses). Nerd Mode is **not** added until there is something to put behind it.

- **Tests:** preferences round trip; name limits (grapheme count, as WP3.2 did); the speaker note carries the
  saved name on the next turn (fake plugin records it).
- **Device checklist:** save a name, send a message, the plugin's note names you (visible in the agent's reply
  if asked "who am I").

### B10. Scheduled tasks — M

New screen, parity with ☰ › Scheduled tasks.

- **Port from:** `DirectHermesScheduledTasksClient.swift`, `Bighelp/Scheduling/`, the `scheduledTasks*` cases in
  `WorkspaceOperation.swift` and their filter rows in `DirectHermesWorkspaceClient.swift`.
- **Wire:** `scheduled_tasks.list / pause / resume / run / update / create / delete / delivery_targets`; events
  `cron.changed`, `scheduled.completed`, `scheduled.failed`; REST `GET /api/cron/blueprints`,
  `POST /api/cron/blueprints/instantiate`, `GET /api/cron/delivery-targets`. Read each shape from the Swift
  client; the roadmap's §2.2 gets the rows in B0.
- **Slice 1:** list, pause, resume, run now, delete (with confirmation), and the list refreshing on
  `cron.changed`. **Slice 2:** create and edit (schedule, prompt, agent, delivery target) and blueprints. A
  blueprint is instantiated only on the person's tap, never on open: no AI spending by default.
- **Tests:** fake host `scheduled_tasks.*` from the stock source (name the files), flow tests per action, "open
  the screen sends only `list`".
- **Device checklist:** pause and resume a real job; run one now and see its run in Chats with "Show scheduled
  runs" on.

### B11. Hardening and the open engineering questions — M

- **Q3 transcript cache bound:** 256 MiB per computer, oldest chats' revisions first, never the open chat; the
  chat list copy is exempt. Log a `transcript.pruned` count only.
- **Q6:** keep `setUnlockedDeviceRequired(false)` while there is no background work; record why.
- **Access credentials staging** (WP2.2 deviation): stage in memory, commit after the first socket opens, as iOS.
- **Backup restore check:** a restored data dir with no vault file shows "Sign in again", never a crash.
- **The 24 h Pixel soak** from the WP3.5 checklist, recorded with the **device** label; its findings become bugs.
- **Lint:** decide `InsecureBaseConfiguration` once (a `lint-baseline.xml` entry with the reason) so 0 warnings
  is the target, not 32.
- **Tests:** prune policy unit tests with virtual files; restore-without-vault flow test.

### B12. Cards: actions and live data — M

- **Read first:** `docs/BIGHELP_CARDS.md` (Live-data policy, Trust boundaries, Visible states), then
  `Chat/ChatCardInteractionEnvironment.swift` and `BighelpCardInteractionStore` (**verified**; interaction state
  is kept under `loopdy.chat-card-interactions.v1.` keys), `GenerativeUICardView.swift`.
- **Build:** form submit, selection send, checklist toggles (local state keyed like iOS so the same card reads
  the same on both), automation pause/resume/run, following exactly how iOS turns an action into a send (read it;
  don't assume `prompt.submit`). `data_sources` stay refused (no live data) unless the doc's policy can be met
  end to end; if not, say so in the log and leave the "This card couldn't be shown" path.
- **Tests:** `CardsFlowTest` per action against the fake host; a card with a source still fails closed.
- **Device checklist:** ask an agent for a small form card; submit it; the reply reflects the fields.

### B13. Usage — M

`usage/list` (`native-provider-usage-v1`, 60 s, 192 KiB) for plans and limits, `usage/activity`
(`native-usage-activity-v1`, 30 s, 1 MiB), and the dashboard's `/api/analytics/usage` and `/api/analytics/models`
for Hermes' own numbers (port from `Bighelp/Usage/`: `ProviderUsageStore`, `UsageStore`, `HostUsageLoader`).
Charts in Material 3 (follow the dataviz rules: one palette, readable in both themes). Made-up numbers in every
fixture and screenshot.

### B14. Workflows — L (later)

Three gates (`native-workflows-v1`, `-edit-v1`, `-trigger-v1`), about 14 routes (status, list, get, draft save,
validate, publish, bind, archive, runs start/control/signoff, templates use/save/delete, pin, unarchive,
trigger/set), 192 KiB request cap. `workflows/status` explains why a computer can't run them
(`chat_runner_missing`). Port from `Bighelp/Workflows/`. Slice 1 is read-only (list, get, runs); sign-off and
editing follow. Needs its own wire inventory before the first slice.

### B15. Kanban — L (later)

A **separate** Hermes plugin, not loopdy: `/api/plugins/kanban/events`, `/api/plugins/kanban/tasks/<id>/attachments`,
`/api/plugins/kanban/attachments/<id>` (**verified** in `Bighelp/Kanban/`). Availability is per computer by
`cacheScopeID` and a failed check during a reconnect is "unknown", not "unavailable" (`KanbanAvailability`,
AGENTS.md). Needs its own wire inventory from `KanbanBoardModel.swift` before the first slice.

### B16. Agents — M (later)

`profiles.describe`, `profiles.get_asset` (avatar) for faces in the chat header and lists; an Agents screen with
the agent's state from real data only (`AgentLiveState`: idle, thinking, replying, has an update). Agent Studio,
templates (`TemplateCatalogStore`, `catalog.bighelp.app`) and the avatar catalog (`avatars.bighelp.app`) stay
deferred; they are large and iOS-art-bound.

### B17. Voice — L (deferred)

Turn-based first (`/api/audio/transcribe`, `/api/audio/speak`, `/api/audio/speak-stream`), then live voice over
`native-voice-v1` (`voice/status|offer|poll|result|close`) with LiveKit's WebRTC. Android Auto only after this.
Read `docs/DIRECT_STREAMING_AND_LIVE_VOICE.md` first.

### Phase 4, Windows

Unchanged from the roadmap (WP4.1–4.5). The one thing to do earlier: keep the `core/` import guard green on every
batch above, since that boundary is the whole Windows plan.

---

## 4. Suggested order and a rough calendar

| Week | Batches | Why this order |
|---|---|---|
| 1 | B0, B1 | Decisions before the next install; device evidence before more features |
| 2–3 | B2, B5 | The chat's promises first; prompts complete the protocol core |
| 3–4 | B3, B4 | Finish what is half built before starting new screens |
| 5–6 | B6, B9, B7a | The app starts to look and feel like bighelp; alerts while open |
| 7–8 | B8, B10, B11 | Parity items and the soak |
| 9+ | B12, B13, then B14/B15/B16 as wanted; B7b when unblocked; B17 last | Larger surfaces, each behind its gate |

Two engineers (or two agents on separate branches) can run B2 ∥ B5, B3 ∥ B6, and B8 ∥ B10 without merge pain:
they touch different packages under `core/` and `ui/`.
