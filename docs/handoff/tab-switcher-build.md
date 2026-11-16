# Handoff: tab-switcher build

Branch: main

Written 2026-11-15 from a session on ripr. The design doc is approved. No code yet.

## Next action

1. Apply the one cross-repo change below to the doc (Phase 2b reuses tiler's extension), in Scott's words if he wants to confirm it.
2. `/plan:build docs/plan/2026-11-15-tab-switcher.md`. Phase 0 needs Scott: AMO signing creds in env (`MOZILLA_JWT_ISSUER` / `MOZILLA_JWT_SECRET`), KB16 key presses, workspace jumps. Walk it one step per message.

## Read first

1. `docs/plan/2026-11-15-tab-switcher.md` (Status: Approved; panel rounds 1-3 folded in; round 3's window redesign was not panel-reviewed because the cap was hit, the two load-bearing claims were verified by the author).
2. Panel run: `~/.cache/review-panel/runs/fMvytP64/synthesis.md`.
3. Precedent: `~/repos/scottidler/second-brain/borg/src/extension/sign.rs`, `install.rs` (`merge_policy`, line ~198; its read path is unsafe, the doc says why).

## Cross-repo change (decided after approval)

- Scott split window tiling into its own repo, `scottidler/tiler`, which ships a generic GNOME Shell extension (`org.scottidler.WindowControl`).
- This doc's Phase 2b should add `ActivateByTitlePrefix` / `ActivateApp` to that extension instead of shipping a separate `org.scottidler.TabSwitcher` one. Ship order: tiler's extension first.
- Not yet edited into the doc.

## Added scope discussed, not in the doc

- `tab-switcher share <conversation>`: put the current tab URL into a Google Messages conversation's compose box (conversation URLs are stable, e.g. `/web/conversations/EgIxNQ`). Scott has not said yes. Ask before adding.
- Snapchat (Chrome web app, window class `chrome-gfgbgjphjkdhefmnmbhogcpckgpapbag-Default`): `tab-switcher app snapchat` raise only; no Chrome extension (Scott: "I dont think we need to have any extension in Chrome").

## Facts to re-check

| Fact | Command |
|---|---|
| `/opt/firefox/distribution/policies.json` absent | `ls /opt/firefox/distribution/` |
| `control-center-static` still owns F13 | `gsettings get org.gnome.settings-daemon.plugins.media-keys control-center-static` |

## Suggested skills

- `plan:build`, `plan:review` (implementation audit), `rust-cli-coder`, `otto`.

## Session-scoped

- Nothing.
