# Changelog

All notable changes to meh are documented here.

## [Unreleased]

### Added
- **`(launcher)` attrs `:show-bins` and `:show-run-command`** — `:show-bins false`
  restricts results to desktop apps only (no PATH executables); `:show-run-command false`
  removes the "run command" literal fallback row. Both default `true` for existing configs.
- **Launcher: arrow key navigation fixed** — `EventControllerKey` now runs in
  `PropagationPhase::Capture` so Up/Down/Enter/Esc are captured before the GtkEntry's
  default handler; arrow navigation now works reliably.
- **Launcher CSS** — `.launcher`, `.launcher-input`, `.launcher-row`, `.launcher-row.selected`,
  `.launcher-name`, `.launcher-desc`, `.launcher-run-prefix` classes added to dark and
  light SCSS themes, giving the popup a readable dark/light background instead of being
  fully transparent.

### Fixed
- **Duplicate deflisten processes** — every script using `cmd | while read; do ...; done`
  spawned the while loop as a visible bash subshell. Fixed with `shopt -s lastpipe` in
  all three bash deflisten scripts (cava-meh, wifi-available, player-meta). The two
  inline `sh -c` deflisten blobs (pacman, calendar) were extracted into proper
  `getPacman-listen.sh` / `calendar-listen.sh` scripts with the same fix.
- **Orphaned scripts survive daemon restart** — `kill_orphaned_scripts` used SIGTERM
  which bash shells blocked in inotifywait kernel waits can ignore. Switched to SIGKILL.
- **Bar launch leaves stale daemon alive** — `bar-launch.sh` now loops up to 2 s waiting
  for `meh ping` to fail before starting a new daemon, then hard-kills any survivor with
  `pkill -9 -x meh`. Prevents dual-daemon situations on repeated launches.
- **Theme switch freezes bar** — `toggle-reverse-theme.sh` was using `pkill meh` + full
  daemon restart on every theme change. Replaced with `meh reload`; CSS reloads in-place
  with no bar flicker or freeze.
- **GTK4 4.10 deprecation warnings in `circular-progress`** — `style_context().add_provider()`
  replaced with `style_context_add_provider_for_display` scoped to a unique CSS class;
  redundant DrawingArea CSS provider removed (draw_func already clears to transparent).
- **tooltip binding never registered when var not yet in scope** — `eval_attr_str`
  for a tooltip attr containing a defpoll var ref returned `None` at window-build
  time (initial poll still running), causing the entire tooltip block including
  `maybe_bind` to be skipped. Changed to `unwrap_or_default()` so the binding is
  always registered; the setter fires on the first real poll value. Matches the
  pattern already used for the `class` binding.
- **orphaned inotifywait processes for /tmp/meh/ triggers** — `kill_orphaned_scripts`
  only matched processes with the config scripts dir in their cmdline. Added `/tmp/meh/`
  as a second match needle so inotifywait processes watching cal_trigger and similar
  files are terminated on daemon restart.
- **deflisten subprocess leak** — listen vars now run for the daemon's lifetime
  without window gating. Killing/restarting on every popup open/close was
  accumulating orphaned grandchild processes (inotifywait, playerctl --follow,
  nmcli monitor, etc.). Subprocesses restart automatically if they die.
- **bar flicker on popup close** — `update_vars()` was calling
  `rebuild_open_windows()` (full window close/reopen) instead of
  `update_bindings()` (reactive, O(bindings)). Now only changed bindings
  are pushed.
- **menus not closing on click-outside** — click-catcher and popup windows
  moved from `stacking "fg"` / `stacking "bottom"` to `stacking "overlay"`,
  ensuring they receive input above app windows on Wayland.
- **deflisten process groups** — spawned with `.process_group(0)` so
  `killpg(SIGTERM)` on shutdown reaches grandchildren, not just the shell
  wrapper.

### Added
- **Native `(launcher)` widget** — instant app search via `gio::AppInfo`,
  PATH executable autocomplete, keyboard nav (↑/↓/Enter/Escape), click-to-launch,
  and a literal "run command" fallback row. No subprocess per keystroke.
- **`dots` CLI** — symlink to `dotbackup`; `dots backup`, `dots restore`,
  `dots check`, `dots list`, `dots prune`, `dots clean` subcommands.
- **PKGBUILD** — Arch Linux package build script.
- **Git repository** — project now tracked in git.

## [0.1.0] — 2026-05-22

Initial release. GTK4 eww fork with:
- Yuck configuration language (ported from elkowar/eww)
- Reactive binding system (ADR-0007): O(bindings) updates, no full tree rebuild
- Poll subprocess gating (ADR-0008): polls paused when no windows open
- Three build profiles: minimal (4.2 MiB), default (6.9 MiB), full
- System tray (opt-in, `systray` feature)
- Declarative animations via `AdwTimedAnimation` (ADR-0009)
- `defsubscribe` for inotify and DBus vars (ADR-0010)
- Granular hot reload — only changed windows are closed/reopened
- Reactive multi-monitor — connect/disconnect handled live
- `(shader)` widget via GtkGLArea (full profile)
