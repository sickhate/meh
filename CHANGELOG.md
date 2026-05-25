# Changelog

All notable changes to meh are documented here.

## [Unreleased]

### Fixed
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
