# Design Document: tab-switcher

**Author:** Scott Idler
**Date:** 2026-11-15
**Status:** Approved (panel rounds 1-3 folded in; round cap reached; nothing open or disputed)
**Review Passes Completed:** 4/5 (draft, correctness, clarity, edge cases)

## Summary

A signed Firefox extension plus a small Rust CLI. A KB16 pad key fires a GNOME shortcut, the CLI hands Firefox an `ext+tabswitch:` URI, and the extension switches to the most recently used tab for that page in the right container, or opens it (pinned) when none exists. The same extension routes navigations into a Work or Home container by URL, so GitHub and Google land in the right account without thinking about it.

## Problem Statement

### Background

- Scott runs Firefox 157 (tarball, `/opt/firefox`) on ripr, Ubuntu GNOME on Wayland. Two profiles in `profiles.ini`; the `[Install...]` section selects `t8hf6jks.default-release` (the one in use), while `n891nkrt.default` carries `Default=1`. No containers in use today.
- A DOIO KB16 macropad (16 keys, 3 knobs, VIA firmware) is plugged into ripr. It can send any keycode, including F13-F24 and modifier combos.
- Two Google personas: work `scott.idler@tatari.tv`, home `scott.a.idler@gmail.com`. Two GitHub personas: work (tatari-tv org) and personal `scottidler`. Work is the primary for both.
- Work SaaS behind Tatari Okta, hosts taken from Firefox history on ripr: `tatari.okta.com` (1176 visits), `wd12.myworkday.com/tatari` (345), `app2.greenhouse.io` (257), `app.drata.com` (165), `app.wiz.io` (59).
- In-house precedent for a signed, self-installed extension: second-brain's `sb borg extension`. `borg/src/extension/sign.rs` runs `web-ext sign --channel unlisted` with `MOZILLA_JWT_ISSUER` / `MOZILLA_JWT_SECRET` from env. `borg/src/extension/install.rs` merges a `force_installed` entry into `policies.json` by extension id (`merge_policy`, line 198). `borg/src/service.rs:431` installs a GNOME custom keybinding via `gsettings`.

### Problem

- Tab sprawl: thousands of tabs across several windows. Every "go to Gmail" opens another tab because `firefox <url>` always creates a new one, and finding the existing tab by hand is slower than opening a new one.
- Wrong-account friction: Google and GitHub sessions are shared across the whole profile. Google's `authuser=` only picks the account at open time (the URL becomes `/mail/u/N/` after load, and N follows sign-in order). GitHub has no URL-level account selector at all: its account switcher flips the whole profile.

### Goals

- One pad key -> one page: switch to the latest existing tab for that page, else open it. (Scott, 2026-11-15: "open a tab to that page IFF it did not exist. otherwise it would find the latest existing and switch to that.")
- Work | Home personas kept apart by Firefox containers, one account per container. (Scott, 2026-11-15: "I might be willing to adopt containers to solve that issue and maybe others.")
- GitHub routed by owner in the URL path: `scottidler` -> Home, everything else -> Work. (Scott, 2026-11-15: "scottidler (personal) in one url and tatari-tv (work) in the other? seems like you could figure it out")
- Google apps per persona: Gmail, Calendar, Drive, Docs, Sheets, Slides, Meet transcripts (work only: transcription is a Workspace feature). (Scott, 2026-11-14)
- Work SaaS on keys: Workday, Wiz, Drata, Greenhouse, extensible. (Scott, 2026-11-15)
- Pad-opened tabs pinned, so they stay findable among hundreds. (Scott asked 2026-11-15 whether container tabs stay pinned; pinning proposed in-session, then this doc requested.)
- Works across Scott's window layout: two or more Firefox windows on workspace 1, extra Firefox windows opened ad hoc, and apps (Telegram, Obsidian) on workspace 2. Selecting something on another workspace switches to that workspace. (Scott, 2026-11-15: "is it going to switch us to that other desktop?")
- Signed and force-installed the same way the borg extension is. (Scott, 2026-11-15: "we write our own extension that I install into Firefox. We sign it.")

### Non-Goals

- Excluded: KB16 keymap programming. VIA does that by hand; this doc only fixes which keycodes the GNOME side listens for.
- Excluded: closing, deduplicating or archiving the existing tab backlog. The extension stops new sprawl; it does not clean old sprawl.
- Excluded: more than one account per container. `/u/0` inside a container is that container's only Google account by construction.
- Excluded: auto-focusing any app on its own attention requests. Activation happens only when Scott presses a key (round 3: an allowlist that raised Telegram on every new message was not asked for).
- Excluded: Chrome / Chromium. VIA needs Chrome, but browsing is Firefox.
- Parked: a native-messaging host so the extension reads `~/.config` live. Revisit if config edits needing a Firefox restart become an observed annoyance (see Addendum A).
- Parked: a knob that cycles through matching tabs. Revisit after keys are in daily use.

## Proposed Solution

### Overview

```
KB16 key (VIA: F14..F19, F24, with Shift/Ctrl)
  -> GNOME custom shortcut (gsettings, user dconf)
  -> tab-switcher open <target> [--as home]        (Rust CLI, reads ~/.config/tab-switcher/tab-switcher.yml)
  -> firefox 'ext+tabswitch:open?url=..&match=..&container=..&pin=1'
  -> extension handler page -> background: switch-or-open -> close handler tab

any navigation (link, address bar, external app)
  -> extension webRequest listener -> route table (from policies.json 3rdparty block)
  -> wrong container? reopen in the right one
```

Change frequency drives the split:

- Targets (what each key opens) change often -> live only in the CLI's YAML and ride inside each URI. No re-sign, no restart.
- Routes and containers change rarely -> delivered to the extension through `policies.json` `3rdparty` -> `browser.storage.managed`. Edit YAML, `sudo tab-switcher install --policy`, restart Firefox (managed storage is read once per run: Firefox 157 `ext-storage.js:428`). No re-sign, because host permissions are `<all_urls>` (below), so a route on a new host needs nothing from the `.xpi`.
- Logic changes rarely -> the signed `.xpi`.

### Architecture

**Extension (`extension/`, plain ES modules, no build step)**

- `manifest.json`: MV2 or MV3 settled in Phase 0 (whichever keeps blocking `webRequest` on Firefox 157). Permissions: `tabs`, `contextualIdentities`, `cookies`, `storage`, `webRequest`, `webRequestBlocking`, host permission `<all_urls>`. `protocol_handlers: [{ protocol: "ext+tabswitch", uriTemplate: "/handler.html#%s" }]`. Fixed gecko id so the policy entry and managed storage key on it. `version` stamped from the CLI's crate version at sign time, copying borg (`borg/src/extension.rs:28`, `sign.rs:24`).
- Why `<all_urls>`: routes are config, and a per-host permission list would make every new route host a re-sign. This is Scott's own extension on his own machine; the listener only acts on URLs a route matches and passes everything else through untouched.
- `handler.html` + `handler.js`: parses the URI, sends one message to the background, nothing else.
- `background.js`: wiring only. Handles `open` messages, `webRequest` routing, and the toolbar button.

**Startup order (background)**

1. Synchronously, at top level: register the blocking `onBeforeRequest` listener and the `runtime.onMessage` listener, and create one `init` promise.
2. `init`: `storage.managed.get()` -> validate `{containers, routes}` -> ensure containers exist (`contextualIdentities.query` by name, `create` if missing) -> build the key -> `cookieStoreId` map.
3. Every listener `await`s `init` before acting. Firefox accepts a Promise from a blocking `onBeforeRequest`, so a cold-start navigation waits for routes instead of racing them.
4. `init` fails (no policy entry: `storage.managed` throws "Managed storage manifest not found", `ext-storage.js:109`; or the config fails validation) -> badge `!`, console error tagged `[tab-switcher]`, routing passes every request through, and every `open` shows the error in its handler tab. Loud, visible, and it never moves a tab on a guess.
- Pure modules, unit-tested with `node --test`, no browser APIs inside:
  - `match.js`: `matches(url, pattern)`. Same scheme and host; pattern path segments are a prefix of URL path segments (`https://github.com/scottidler` matches `/scottidler` and `/scottidler/bump`, not `/scottidler-old`). Query and fragment ignored.
  - `pick.js`: `pickTab(tabs, {match, cookieStoreId}, focusTimes)` -> the matching tab with the latest focus time (from the background's map, falling back to `lastAccessed`), or `null`.
  - `route.js`: `routeFor(url, currentStoreId, routes)` -> `null` (leave it) or the target container key. First matching route wins; `assign` routes always apply, `default` routes apply only from a no-container tab.

**Switch-or-open (background)**

1. Resolve container key -> `cookieStoreId`.
2. `tabs.query({})` across all normal windows, then `pickTab`.
3. Found -> activate it and mark its window with the nonce title preface; the CLI has GNOME Shell raise it (see Windows and workspaces). Tabs are never moved between windows: the layout across Scott's windows stays his.
4. Not found -> `tabs.create({url, cookieStoreId, pinned: pin, windowId: <handler tab's window>})`.
5. Close the handler tab.

Edge rules:

- Only tabs in the target container count. The existing no-container backlog (Gmail tabs signed in before containers) never matches, so the first press opens a fresh Work tab. Intended: the backlog is a non-goal.
- Two presses before the first tab exists: `tabs.create` returns before the page loads (Firefox 157 `ext-tabs.js:823-828`), and a Google tab may still sit on `accounts.google.com`, so a second `tabs.query` would miss it. The background keeps an in-flight map `(match, container) -> tabId`, set on create and cleared when that tab finishes loading a matching URL, closes, or fails. A press whose key is in the map switches to that tabId.
- Pinning: `pin` defaults to `false`. A found tab is switched to as-is, never re-pinned.
- Discarded tabs (Auto Tab Discard is installed) keep `lastAccessed`; activating one reloads it. No special case.
- A container deleted by hand is recreated at startup with a new `cookieStoreId`; that container then needs its account signed in again.

**Routing (background)**

Scope: top-level document loads (`main_frame`), GET only. Everything else passes through. GitHub's in-page (Turbo / history API) navigations make no `main_frame` request, so a click from a Home repo to a tatari-tv repo inside GitHub can stay in Home; Phase 0 (f) measures how often, and the toolbar button is the fix when it happens.

Decision per request, in order (each step "pass" = return nothing, the request proceeds). Decisions are per `requestId`, which Firefox keeps across a redirect chain (`WebRequest.sys.mjs:1183-1194`): a request that passed at step 2 or 3 keeps passing through every redirect, and the record clears on `onCompleted` or `onErrorOccurred`. Without this, a restored Gmail tab that bounces through `accounts.google.com` would fail the step-3 check on its second hop.

1. Not GET, private window, or init failed -> pass.
2. Tab carries a one-shot exemption (set by the toolbar button on the tab it created) -> move the exemption from the tab to this `requestId`, pass. It covers the whole redirect chain of that first load and nothing after it.
3. Request URL equals the tab's current `url`, both compared with the fragment stripped (a reload, a discarded tab waking up, a session restore) -> pass. This keeps the no-container backlog untouched: old Gmail tabs reload where they are.
4. `routeFor(url, tab.cookieStoreId, routes)` returns `null` -> pass.
5. Otherwise return `{cancel: true}` and open the URL in the target container with `tabs.create({url, cookieStoreId, windowId, index, active, pinned, openerTabId})` copied from the original tab. Then remove the original only if it is disposable: its `url` is `about:blank`, `about:newtab` or `about:home`. A tab that has not committed its first load still reports `about:blank` (`ext-tabs-base.js:245`), which covers external opens from Slack, the terminal and the CLI; Phase 0 (g) confirms it. Anything else stays, so a cancelled click leaves the page it came from.

Route modes:

- `assign`: applies from any container. Used only for owner paths and for single-tenant work hosts (Okta, Workday, Greenhouse, Drata, Wiz).
- `default`: applies only from a no-container tab (and then only to disposable tabs or new navigations, per step 3 and 5). Used for shared hosts: GitHub's non-owner paths and the Google apps.

GitHub invariant: `/scottidler/...` assigns Home and `/tatari-tv/...` assigns Work. Every other GitHub path (`/login`, `/session`, `/logout`, `/settings`, `/notifications`, `/pulls`, gists, OAuth, other orgs' repos) is a `default` route: it stays in whatever container the tab is already in, and only a no-container tab goes to Work. Walkthrough of the hardest case: signed out in Home, open `github.com/scottidler/bump` -> GitHub redirects to `/login` -> a `default` route in a Home tab -> stays in Home -> sign in -> back to `/scottidler/bump` in Home. Auth never leaves Home.

**Windows and workspaces**

Facts on ripr (2026-11-15): 4 fixed workspaces, `focus-new-windows` = `smart`; GNOME Shell extensions already in use, two of which export session D-Bus interfaces for this kind of job (`wispr-flow-window-bridge@wispr.flow`: `org.wispr.flow.WindowBridge` with `GetFocusedWindow`/`GetWindowList`; `vibe-typer-focus@vibetyper.com`). GNOME's media-keys daemon launches custom shortcuts with no startup token (`/usr/libexec/gsd-media-keys` uses a plain `g_app_launch_context_new`; panel round 3), so Firefox has no activation token to pass on, and `windows.update({focused})` alone cannot switch workspace.

Design: the window raise is done by GNOME Shell, on request from the CLI, only when a key is pressed.

- **Shell extension** (`gnome-shell/`, installed by `install --shortcuts`): exports `org.scottidler.TabSwitcher` on the session bus with two methods, following the wispr bridge's shape:
  - `ActivateByTitlePrefix(prefix, timeout_ms) -> bool`: polls the window list for up to `timeout_ms` for a window whose title starts with `prefix`, then `Main.activateWindow` (switches workspace, focuses). Returns false on timeout.
  - `ActivateApp(wm_class) -> bool`: activates the most recently used window with that `wm_class`; false if none.
- **Firefox target flow:**
  1. CLI makes a nonce `ts-<random>` and puts it in the URI.
  2. Extension picks the tab, `tabs.update(id, {active: true})`, then `windows.update(windowId, {titlePreface: "[ts-<random>] "})`, then closes the handler tab.
  3. CLI calls `ActivateByTitlePrefix("[ts-<random>]", 1500)`. Shell raises that window on whatever workspace it lives on.
  4. Extension clears the title preface on the next `windows.onFocusChanged` to that window, or after 3s.
- **Native app flow (Telegram, Obsidian):** `tab-switcher app <name>` -> `ActivateApp(<wm_class>)`; if it returns false, run the app's configured launch command. The daily note key runs `ActivateApp` for Obsidian, then `xdg-open obsidian://daily?vault=obsidian`.
- **The handler tab's own window:** with no token, Firefox's own focus request for the handler tab is refused, so Scott is not dragged through that window's workspace. GNOME may show "Firefox is ready" for it; Phase 0 (b) records whether it does and which signal fired (`window-demands-attention` vs `window-marked-urgent`, both handled by `windowAttentionHandler.js` in Shell 50.1).
- Verified by the author after round 3 (the panel cap is 3, so the redesign got no round 4): Firefox 157's `windows.json` schema in `/opt/firefox/browser/omni.ja` carries `titlePreface`; GNOME Shell's `Main.activateWindow` (`/usr/lib/gnome-shell/libshell-18.so`, `ui/main.js:920-933`) calls `workspace.activate_with_focus` when the window is on another workspace. How fast a title preface reaches the Wayland title is unmeasured; the 1500ms poll covers it and Phase 0 (b) measures it.
- **Recency across windows:** `lastAccessed` cannot rank across windows, because Firefox reports `Date.now()` for every window's selected tab (`tabbrowser/tab.js:321-322`), so a stale background window always wins. The background keeps its own `tabId -> last focused time`, updated on `tabs.onActivated` and `windows.onFocusChanged`, stored in `storage.session` so a background restart keeps it. `pickTab` ranks by that map and falls back to `lastAccessed` only for tabs never seen focused since Firefox started.

**Toolbar button: "Reopen in Work | Home"**

- Opens the current URL in the other container and closes the current tab. The escape hatch for a Google link that `default` routing put in Work but belongs to Home.
- The tab it creates carries a one-shot routing exemption (step 2 above), so an `assign` route cannot bounce it straight back. The next navigation in that tab routes normally.

**CLI (`tab-switcher`, Rust, thin clap shim over a lib crate)**

- `tab-switcher open <target> [--as <container>]`: builds the URI with a fresh nonce, runs `firefox <uri>`, then calls `ActivateByTitlePrefix`. `--as` must be in the target's `as` list, loud error otherwise. A false return exits nonzero with "window not raised".
- `tab-switcher app <name>`: `ActivateApp(<wm_class>)`, else the app's `launch` command.
- `tab-switcher install --shortcuts`: writes one GNOME custom keybinding per `keys:` entry under `/org/gnome/settings-daemon/plugins/media-keys/custom-keybindings/tab-switcher-<n>/`, same `gsettings` mechanics as borg's `install_hotkey`. Idempotent; removes its own stale entries, never touches others.
- `tab-switcher install --policy` (root): merges into `/opt/firefox/distribution/policies.json`. This is new code, not a copy of borg: borg's `merge_policy` (`install.rs:198`) only touches `ExtensionSettings`, and its read path turns any parse error into `{}` (`install.rs:447`), which would clobber a malformed file. Rules:
  - File missing -> start from `{"policies": {}}`. File present but unreadable, unparseable, or with a non-object `policies` / `ExtensionSettings` / `3rdparty` / `3rdparty.Extensions` (Firefox requires an object: `policies-schema.json:49-58`) -> abort, nonzero exit, file untouched. Same checks for `--remove`.
  - Set only `policies.ExtensionSettings[<id>]` (force_installed, `install_url` = `file://` to the signed `.xpi`) and `policies["3rdparty"].Extensions[<id>]` = `{containers, routes}`. Every other key and value, including other extensions' entries under both, is preserved byte-for-byte in meaning.
  - Write to a temp file in the same directory, then rename (atomic).
  - Rollback for a broken router: remove `<id>` from both maps (`tab-switcher install --policy --remove`) and restart Firefox.
- Sign: `otto sign` runs `web-ext sign --channel unlisted` with the same `MOZILLA_JWT_ISSUER` / `MOZILLA_JWT_SECRET` env the borg path uses.

### Data Model

`~/.config/tab-switcher/tab-switcher.yml` (repo ships one annotated example):

```yaml
containers:
  work: { name: Work, color: blue, icon: briefcase }
  home: { name: Home, color: green, icon: fingerprint }

targets:
  gmail:     { url: "https://mail.google.com/mail/u/0/", match: "https://mail.google.com/mail", as: [work, home], pin: true }
  calendar:  { url: "https://calendar.google.com/calendar/u/0/r", match: "https://calendar.google.com/calendar", as: [work, home], pin: true }
  drive:     { url: "https://drive.google.com/drive/u/0/my-drive", match: "https://drive.google.com/drive", as: [work, home] }
  docs:      { url: "https://docs.google.com/document/u/0/", match: "https://docs.google.com/document", as: [work, home] }
  sheets:    { url: "https://docs.google.com/spreadsheets/u/0/", match: "https://docs.google.com/spreadsheets", as: [work, home] }
  slides:    { url: "https://docs.google.com/presentation/u/0/", match: "https://docs.google.com/presentation", as: [work, home] }
  transcripts: { url: "https://drive.google.com/drive/u/0/search?q=transcript", match: "https://drive.google.com/drive/u/0/search", as: [work] }
  github:    { url: "https://github.com/tatari-tv", match: "https://github.com/tatari-tv", as: [work] }
  github-me: { url: "https://github.com/scottidler", match: "https://github.com/scottidler", as: [home] }
  workday:   { url: "https://wd12.myworkday.com/tatari", match: "https://wd12.myworkday.com/tatari", as: [work], pin: true }
  greenhouse: { url: "https://app2.greenhouse.io/dashboard", match: "https://app2.greenhouse.io", as: [work] }
  drata:     { url: "https://app.drata.com/dashboard", match: "https://app.drata.com", as: [work] }
  wiz:       { url: "https://app.wiz.io/", match: "https://app.wiz.io", as: [work] }

routes:            # first match wins
  - { match: "https://github.com/scottidler", to: home, mode: assign }
  - { match: "https://github.com/tatari-tv", to: work, mode: assign }
  - { match: "https://github.com", to: work, mode: default }
  - { match: "https://mail.google.com", to: work, mode: default }
  - { match: "https://calendar.google.com", to: work, mode: default }
  - { match: "https://drive.google.com", to: work, mode: default }
  - { match: "https://docs.google.com", to: work, mode: default }
  - { match: "https://tatari.okta.com", to: work, mode: assign }
  - { match: "https://wd12.myworkday.com", to: work, mode: assign }
  - { match: "https://app2.greenhouse.io", to: work, mode: assign }
  - { match: "https://app.drata.com", to: work, mode: assign }
  - { match: "https://app.wiz.io", to: work, mode: assign }

apps:              # wm_class values are recorded by Phase 0 (b); desktop files declare these
  obsidian: { wm-class: "md.obsidian.Obsidian", launch: "gtk-launch md.obsidian.Obsidian" }
  telegram: { wm-class: "TelegramDesktop", launch: "gtk-launch <telegram desktop id, hash suffix included>" }

keys:              # GNOME accelerator -> CLI args
  - { key: "F14",       run: "open gmail" }
  - { key: "<Shift>F14", run: "open gmail --as home" }
```

- First target `as` entry is the default container.
- `match` defaults to `url` when omitted. `pin` defaults to `false`.
- Unknown keys, unknown container names, a `--as` outside `as`, or a `key` on a reserved keysym (below) -> load error, nonzero exit. No silent defaults.

**Reserved keysyms (observed on ripr, `/usr/share/X11/xkb/symbols/inet`, `gsettings`)**

- F13 -> `XF86Tools`, bound by GNOME `control-center-static` (opens Settings). Unusable.
- F20 -> `XF86AudioMicMute`, F21-F23 -> touchpad toggles / `XF86Assistant`. Unusable.
- Usable plain: F14-F18 (emit `XF86Launch5`-`XF86Launch9`), F19, F24. Seven keys, times Shift / Ctrl / Alt variants.
- GNOME matches the keysym the key emits, not its F-number: a pad key set to F14 emits `XF86Launch5`, so a GNOME binding on `F14` never fires. Config names keys by pad F-number (what Scott types into VIA); `install --shortcuts` translates through this table (F14->`XF86Launch5` ... F18->`XF86Launch9`, F19->`F19`, F24->`F24`). Phase 0 (e) verifies the table.

### API Design

URI handed to Firefox (all values percent-encoded):

```
ext+tabswitch:open?url=<url>&match=<pattern>&container=<key>&pin=<0|1>&nonce=<ts-random>
```

- The extension trusts nothing in it beyond those five fields (`nonce` must match `ts-[a-z0-9]{12}`): `container` must name a configured container, `url` and `match` must be `https:`. Anything else -> the handler shows the error in its own tab and does not close it (loud, visible failure).

### Implementation Plan

#### Phase 0: Prove the four environmental assumptions (spike, nothing committed)
**Model:** opus
- Throwaway extension loaded via `about:debugging`, fixed gecko id, with the `ext+tabswitch` handler and a background that logs.
- (a) A GNOME custom shortcut on `XF86Launch5` running `firefox 'ext+tabswitch:open?...'` reaches the handler with no "choose application" prompt, with Firefox running. Cold start (Firefox closed) cannot use an `about:debugging` add-on, which is gone after restart: sign the spike unlisted (`web-ext sign`) and force-install it by policy for this probe only. This also covers the first launch after a fresh force-install, when Firefox has no saved blocking listener to start early (`ext-webRequest.js:139-147`).
- (b) Load a throwaway GNOME Shell extension exporting `ActivateByTitlePrefix` and `ActivateApp`. With Scott on workspace 1 and the target in (1) another Firefox window on workspace 1, (2) a Firefox window on workspace 2, (3) Obsidian on workspace 2, (4) Telegram on workspace 2: press the key and record every workspace Scott passes through, whether any "is ready" notice appears and which signal fired, and the runtime `wm_class` of Obsidian and Telegram (`GetWindowList` on the wispr bridge reports it today). Pass = one jump, straight to the target.
- (c) A `3rdparty.Extensions[<id>]` block in `/opt/firefox/distribution/policies.json` shows up in `browser.storage.managed.get()` for that id after restart.
- (d) Blocking `webRequest.onBeforeRequest` on `main_frame` works under the manifest version chosen; record MV2 vs MV3.
- (e) KB16 set to F14 in VIA fires a GNOME shortcut bound to `XF86Launch5`, and one set to F19 fires a binding on `F19`.
- (g) `firefox https://github.com/scottidler` from a terminal: record the `tab.url` the background sees at `onBeforeRequest`. Expected `about:blank`; any other value means the disposable test needs revising before Phase 3.
- (f) From a Home tab on `github.com/scottidler`, click through to a `tatari-tv` repo inside GitHub: record whether a `main_frame` request fires (routing applies) or GitHub navigates in-page (it stays in Home).
- **Success criteria:** each of (a)-(g) has a recorded yes/no in this doc's Resolved Decisions; any "no" becomes a doc revision before Phase 1.

#### Phase 1: Repo scaffold and pure modules
**Model:** sonnet
- Repo `scottidler/tab-switcher`: cargo workspace (lib + bin), `extension/`, `otto.yml` with `ci` = `cargo test` + `cargo clippy -- -D warnings` + `node --test extension/` + `web-ext lint -s extension`.
- `match.js`, `pick.js`, `route.js` with tests, positive and negative per rule (segment boundary, `assign` vs `default`, first-match-wins, `lastAccessed` tie, the step 1-5 routing decision including reload pass-through (fragment stripped), per-requestId carry across a redirect, and the disposable test, URI round-trip of a URL carrying a query and a fragment).
- `bin/count-tabs <match> <work|home|none>`: the live oracle for Phases 2-5, so it ships here. Resolves the profile from the `[Install...]` section of `profiles.ini` (or `--profile <path>`), waits up to 30s for a `sessionstore-backups/recovery.jsonlz4` with an mtime after its own start, then prints the count. `none` = tabs in no container. Runs via `uv run --with lz4`; test tool only.
- **Success criteria:** `otto ci` exits 0; deleting the segment-boundary check in `match.js` makes `node --test extension/` fail.

#### Phase 2: Switch-or-open
**Model:** opus
- `handler.html/js`, startup order and `init`, background open path, in-flight map, container ensure, pinning, the Phase 0 focus mechanism, URI validation with visible error, failure badge.
- **Success criteria:** with the unsigned build in `about:debugging`, running the `firefox 'ext+tabswitch:open?...gmail...'` command twice leaves exactly one Work Gmail tab (`bin/count-tabs`).

#### Phase 2b: GNOME Shell activation extension
**Model:** sonnet
- `gnome-shell/` extension exporting `org.scottidler.TabSwitcher` (`ActivateByTitlePrefix`, `ActivateApp`), modelled on `wispr-flow-window-bridge`. CLI calls from `open` and `app`; title preface set and cleared by the Firefox extension.
- **Success criteria:** the four Phase 0 (b) cases each produce one workspace jump straight to the target; `gdbus call --session --dest org.gnome.Shell --object-path /org/scottidler/TabSwitcher --method org.scottidler.TabSwitcher.ActivateApp nope` returns `(false,)`.

#### Phase 3: Routing and the toolbar button
**Model:** opus
- `webRequest` routing (steps 1-5), GitHub invariant routes, one-shot exemption, "Reopen in Work | Home" button.
- **Success criteria:** starting with no `scottidler` tabs, `firefox https://github.com/scottidler/bump` from a terminal gives `bin/count-tabs https://github.com/scottidler home` = 1, `work` = 0, `none` = 0.

#### Phase 4: CLI
**Model:** sonnet
- Config load and validation, `open`, `install --shortcuts`, `install --policy` (merge rules above, `--remove`), annotated example config.
- **Success criteria:** `tab-switcher open gmail --as nope` exits nonzero naming the allowed list; a config with `key: F13` fails to load naming `control-center-static`; `install --policy` on a fixture holding a foreign extension's `ExtensionSettings` entry and `3rdparty` entry leaves both values equal (`jq` compare), and on a malformed fixture (unparseable JSON, and one with `3rdparty.Extensions` = `[]`) exits nonzero with the file's sha256 unchanged, for both install and `--remove`.

#### Phase 5: Sign, install, verify live on ripr
**Model:** sonnet
- `otto sign`, `sudo tab-switcher install --policy`, `tab-switcher install --shortcuts`, restart Firefox, sign each account into its container once, set the KB16 keys in VIA.
- **Success criteria:** the Acceptance Criteria below, run on ripr.

## Acceptance Criteria

- [ ] Starting state: exactly one Work Gmail tab, in a window that is not focused (`bin/count-tabs https://mail.google.com/mail work` = 1). Press the Gmail key twice: the count is still `1` and that tab is the active tab of the focused window.
  Observed on main: not runnable, no repo yet; depends on Phases 2 and 4.
- [ ] Starting with no `scottidler` tabs, `firefox https://github.com/scottidler/bump` from a terminal: `bin/count-tabs https://github.com/scottidler home` = 1, `work` = 0, `none` = 0.
  Observed on main: not runnable; depends on Phases 3 and 4.
- [ ] After `sudo tab-switcher install --policy`, `jq '.policies.ExtensionSettings | keys' /opt/firefox/distribution/policies.json` lists the tab-switcher id, and `jq -S 'del(.policies.ExtensionSettings["<id>"], .policies["3rdparty"].Extensions["<id>"])'` of the file equals the same filter applied to the pre-install copy (every foreign value preserved).
  Observed on main (2026-11-15): `/opt/firefox/distribution/policies.json` does not exist on ripr; only `distribution.ini` is there.
- [ ] Starting state: Scott on workspace 1, the only Work Gmail tab in a Firefox window on workspace 2. Press the Gmail key. Then `gdbus call --session --dest org.gnome.Shell --object-path /org/wispr/flow/WindowBridge --method org.wispr.flow.WindowBridge.GetFocusedWindow` reports `"wmClass":"firefox"` and a title containing `Gmail` (a focused window is on the active workspace, so this proves the jump), and `bin/count-tabs https://mail.google.com/mail work` is still `1`.
  Observed on main (2026-11-15): the oracle works today, returning `{"appId":"firefox.desktop",...,"wmClass":"firefox"}` for the focused Firefox window; the key press depends on Phases 2, 2b and 4.
- [ ] `gsettings get org.gnome.settings-daemon.plugins.media-keys custom-keybindings` lists the `tab-switcher-*` paths, and `gsettings get org.gnome.settings-daemon.plugins.media-keys control-center-static` still prints `['XF86Tools']`.
  Observed on main (2026-11-15): `control-center-static` prints `['XF86Tools']`; no `tab-switcher-*` entries (none expected yet).

## Resolved Decisions

- 2026-11-15, Scott: containers adopted for Work | Home.
- 2026-11-15, Scott: own extension, signed like the borg extension, force-installed via policy.
- 2026-11-15, author (from `gsettings` + xkb on ripr): F13 and F20-F23 are reserved; keys use F14-F19, F24 plus modifiers.
- 2026-11-15, panel round 1 (both seats, no objection): repo `scottidler/tab-switcher`.
- 2026-11-15, panel round 1: routes ride `3rdparty` -> `storage.managed`; host permission is `<all_urls>` so routes never force a re-sign; missing policy is a loud init failure.
- 2026-11-15, panel round 1: routing is GET `main_frame` only, cancels in both branches, removes only disposable tabs, skips reloads/restores, and assigns only owner paths and single-tenant work hosts.
- 2026-11-15, Scott (after round 2): multiple Firefox windows and workspace 2 apps must be handled; `tabs.move` dropped as a fallback because the handler's window can sit on the wrong workspace.
- 2026-11-15, panel round 3: no activation token reaches Firefox from a GNOME shortcut, so window raising moves to a key-gated GNOME Shell D-Bus extension (`ActivateByTitlePrefix` via a nonce title preface, `ActivateApp` by `wm_class`); the attention-allowlist helper is dropped (it would raise Telegram on every message); cross-window recency is tracked by the background because `lastAccessed` is `Date.now()` for every selected tab.
- 2026-11-15, panel round 2: routing decisions are keyed by `requestId` across redirects; the disposable test is `about:blank`/`newtab`/`home` only (the 2s age clause dropped: it could close a page that loaded and then got a fast click); `3rdparty.Extensions` must be an object. All five round-1 pushbacks accepted by both seats.
- 2026-11-15, panel round 1, rejected: the architect's `webNavigation` fix for "no history" (that API exposes no history length); reconciling existing tabs on route change (no observed need); a rollback plan beyond `--remove` + restart.

## Alternatives Considered

### Alternative 1: `firefox --new-tab <url>` from a script (the `goog` script in the planner page)
- **Description:** pad key opens the URL with `authuser=`.
- **Pros:** no extension.
- **Cons:** a new tab on every press, which is the sprawl. No GitHub persona.
- **Why not chosen:** fails the switch-to-existing goal.

### Alternative 2: Firefox `commands` keyboard shortcuts inside the extension
- **Description:** pad sends a key combo that the extension itself binds.
- **Pros:** no CLI, no GNOME shortcuts.
- **Cons:** only fires while Firefox has focus.
- **Why not chosen:** the keys must work from the terminal and Slack.

### Alternative 3: Mozilla Multi-Account Containers "Always open in"
- **Description:** existing add-on, per-host container assignment.
- **Pros:** maintained by Mozilla.
- **Cons:** per host, not per path, so `github.com/scottidler` vs `github.com/tatari-tv` cannot be split; no switch-or-open.
- **Why not chosen:** misses both core goals.

### Alternative 4: Second Firefox profile for Home
- **Pros:** total isolation.
- **Cons:** two browser processes, two window sets, links from apps land in one profile only.
- **Why not chosen:** containers give the same cookie isolation inside one tab strip.

### Alternative 5: Generalize `sb borg extension` to manage N extensions
- **Pros:** one signer and installer.
- **Cons:** couples a tab tool to the second-brain release cycle.
- **Why not chosen:** copy the pattern, not the crate. Revisit if a third extension shows up.

## Technical Considerations

### Dependencies
- `web-ext` (mise, present on ripr), Node 26 for tests, Rust stable.
- AMO API credentials already used by the borg sign path.

### Performance
- `tabs.query({})` over thousands of tabs per key press. Fine at this scale; measured in Phase 2 against Scott's live profile.

### Security
- The `ext+tabswitch:` scheme is callable by any web page link. The handler only accepts `https:` URLs and configured containers, so the worst a hostile link can do is open or focus an https page in Work or Home, which any link can already do.
- `policies.json` is root-owned; `install --policy` needs sudo, same as `sudo sb bootstrap --extension`.
- No secrets in the repo; AMO credentials stay in env.

### Testing Strategy
- Pure modules: `node --test`, positive + negative per rule, break-the-code check per Phase 1.
- CLI: cargo unit tests for config validation and policy merge (fixture JSON with a foreign extension entry).
- Live: `bin/count-tabs` against the session store on ripr.

### Rollout Plan
- Phase 5 on ripr only. lappy has no pad.

## Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| GNOME Shell API change breaks the activation extension on upgrade | Med | Med | Same exposure as the wispr and superwhisper extensions already running; `open` exits nonzero on a false return, so a break is loud |
| "Firefox is ready" notice on every press | Med | Low | Phase 0 (b) records it; suppressing it is a follow-up only if it shows up |
| Blocking `webRequest` unavailable under chosen manifest | Low | High | Phase 0 (d) decides MV2 vs MV3 |
| Google sign-in redirect bounced between containers | Med | Med | `accounts.google.com` is not routed; only app hosts are |
| Okta session split across containers | Med | Low | Okta and all Tatari SaaS `assign` to Work |
| `policies.json` change needs Firefox restart | High | Low | Only routes and containers live there; targets never do |
| GitHub in-page navigation skips routing | Med | Low | Phase 0 (f) measures it; toolbar button fixes a stray tab |
| Router misbehaves after force-install | Low | High | `install --policy --remove` + restart; init failure already passes all traffic through |

## Open Questions

None.

## References

- `~/repos/scottidler/second-brain/borg/src/extension/sign.rs`, `install.rs` (`merge_policy`, line 198)
- `~/repos/scottidler/second-brain/borg/src/service.rs:431` (GNOME keybinding install)
- KB16 planner artifact: https://claude.ai/artifact/GNCmuBXVmRtDMM1cgSDcfY
- Prior session `28e5895e` (KB16 research sweep, 2026-11-04/05)

## Addendum A: native messaging host (parked)
- A host binary registered with Firefox would let the extension read `tab-switcher.yml` live, removing the restart after route edits.
- Costs a second install surface (`~/.mozilla/native-messaging-hosts/` manifest) and a process per message.
- Parked until restart-after-edit is an observed annoyance.
