# meh

> **meh** = eww + GTK4. A modern, Wayland-only, performance-first reimagining
> of elkowar/eww as a status bar and widget system.
>
> Binary: `meh`. Config dir: `~/.config/meh/`.
> meh2 fork: `~/Projects/meh2`.

-----

## Current state (last updated 2026-05-29)

**Phase 1 complete. Phase 2 complete (all 5 items done 2026-05-24).**

Widget verification config is at `examples/widget-test/` — run with
`meh --config ~/Projects/meh/examples/widget-test daemon` then
`meh --config ~/Projects/meh/examples/widget-test open widget-test`.

To test systray: add `(systray)` to a window in your yuck config. The widget
shows one button per running tray item; left-click activates it. Right-click
menus and icon-change signals are Phase 2 work.

Everything else in Phase 1 is complete:
- Three build profiles: minimal=4.2 MiB, default=6.9 MiB, full=6.9 MiB
- Reactive binding system (ADR-0007): `BINDING_COLLECTOR` thread-local, `Binding`
  structs, `update_bindings()` — O(bindings) updates, no full tree rebuild
- Poll subprocess gating (ADR-0008): polls paused when no windows open; daemon
  idle CPU is ~0.17% (static bar) / ~0.35% (1s clock poll)
- CI at `.github/workflows/ci.yml`; minimal-bar example at `examples/minimal-bar/`
- `jq` and `tz` features gate the jaq and chrono-tz link-time cost (saves ~2.7 MiB
  from the minimal build)

Known outstanding issues:
- `hostname` command not in PATH on this machine — HOSTNAME shows "" in state
- `minimal` binary target was 5 MiB; achieved 4.2 MiB
- **Binding update optimised.** `Binding::update()` and `LoopBinding::update()` no longer
  clone the entire `global_vars` HashMap on every frame. They build a minimal var map from
  each expression's var refs, cutting allocation churn that previously inflated RSS from
  ~48 MB to 60–100 MB.

-----

## Read this first

You are an agent working in the **meh** repository. This file is the single
source of truth for what this project is, what it is building toward, and the
rules that govern every change. Read it top-to-bottom at the start of every
session. If anything you are about to do contradicts this file, stop and ask.

Remember: **meh2 is the active fork** with Rhai scripting, plugins, and
meh2-specific config. Anything that is pure meh (no Rhai dependency) should
be fixed here and cherry-picked into meh2. Anything that is meh2-specific
goes in meh2's AGENTS.md.

-----

## Table of contents

1. [What meh is, in one paragraph](#what-meh-is-in-one-paragraph)
2. [The prime directive](#the-prime-directive)
3. [Hard scope](#hard-scope)
4. [Lineage and where to fork from](#lineage-and-where-to-fork-from)
5. [Background — how we got here](#background--how-we-got-here)
6. [Features inherited from eww](#features-inherited-from-eww)
7. [Architecture](#architecture)
8. [Architecture decisions (ADRs)](#architecture-decisions-adrs)
9. [Build profiles and Cargo features](#build-profiles-and-cargo-features)
10. [Coding conventions](#coding-conventions)
11. [Performance principles](#performance-principles)
12. [Roadmap](#roadmap)
13. [Rules for the agent](#rules-for-the-agent)
14. [Getting started — step by step](#getting-started--step-by-step)
15. [Agent setup](#agent-setup)
16. [First-session prompt](#first-session-prompt)

-----

## What meh is, in one paragraph

meh is a **Wayland-only** widget system for status bars, popups, dashboards, and
desktop overlays.  It reads a Lisp-like config language (yuck) describing windows,
widgets, and their reactive data bindings.  Under the hood it is a **GTK4** process
that renders these descriptions into live UI and re-renders only the parts whose
data dependencies change.  It is the spiritual successor to **elkowar/eww** —
same config language, same architecture, but GTK4-native instead of wedged into
the increasingly-unstable GTK3 Layer Shell ecosystem.

The name is short for "maybe eww" — a placeholder that stuck.

-----

## The prime directive

**meh must be lighter, faster, and more maintainable than the eww it
replaces, while keeping full yuck compatibility.**

Concretely:

- A meh config must render identically to the same yuck config on eww.
- A meh binary must be smaller than the equivalent eww build (GTK3 vs GTK4).
- meh idle CPU must be <= eww at equivalent config complexity.
- A feature added to meh must have a clear justification over eww (maintainability
  or performance). If it adds complexity without a clear win, it doesn't go in.

**meh is meant to be minimal, focused, and boring.** It is not a platform. It
is a bar that works, and keeps working.

-----

## Hard scope

**In scope:**
- Full yuck compatibility with eww (same syntax, same behaviour).
- Every widget, attribute, and data source that eww supports.
- Reactive bindings, poll/listen/subscribe variables.
- Layer shell positioning, multi-monitor support.
- System tray (StatusNotifierItem protocol).
- Performance and binary size improvements over eww.
- GTK4-only, plain Wayland-only (no XWayland).

**Out of scope — permanently:**
- X11 support. Wayland-only.
- GTK3 support. GTK4-only.
- CSS dialects beyond SCSS/grass.
- Any scripting language in the data plane (Rhai, Lua, Python, etc.) — see meh2.
- Plugin systems — see meh2.
- Features that exist only because "eww had them" — each feature carries its
  own justification.

**What meh delegates to meh2:**
- Rhai scripting engine for poll/listen sources.
- Rhai event handlers.
- Plugin system.
- Hybrid Rhai+yuck widget configuration.

If a task implies X11 / GTK3 / scripting / plugins, **stop and ask** whether
it belongs in meh or meh2.

-----

## Lineage and where to fork from

```
elkowar/eww          (GTK3, original)
│
└── Ewwii-sh/ewwii   (GTK4, community fork from 2023, active 2023-2024)
    │
    └── sickhate/meh  (this repo — pure GTK4 eww successor, no scripting)
        │
        └── sickhate/meh2  (adds Rhai scripting, plugins, hybrid config)
```

**This repo (`meh`) is the baseline.** meh2 cherry-picks bugfixes from meh.

**Never fork from Ewwii directly.** Always branch from meh.

**Never merge meh2 into meh.** Cherry-pick individual commits.

Upstream clones (for reference, not modification):
```
git clone https://github.com/elkowar/eww          eww-upstream
git clone -b gtk4 https://github.com/elkowar/eww  eww-gtk4-branch
git clone https://github.com/Ewwii-sh/ewwii       ewwii-upstream
```

-----

## Background — how we got here

1. **eww (GTK3)** works but is increasingly unstable on modern Wayland compositors.
   The GTK3 Layer Shell protocol implementation has known bugs on:
   - river: negative layer-surface dimensions crash the compositor
   - Hyprland: interactive `eww update` calls have a ~50-100 ms latency spike
   - general: GTK3 deprecation pressure grows every release cycle

2. **Ewwii** (a community GTK4 fork from 2023) solved the GTK3 problem but introduced
   code churn from the automated GTK3→GTK4 migration (sed replace of callback names
   and type signatures).  The result is functional but harder to maintain than either
   original.

3. **meh** is a from-scratch rewrite of the GTK4 widget implementation that:
   - Keeps Ewwii's architecture (yuck parser from eww, VarState, IPC protocol)
   - Replaces the GTK4 widget layer with clean, idiomatic GTK4 code
   - Uses `libadwaita` for declarative animations
   - Drops X11 entirely (was a dead code path in eww/ewwii)
   - Is ~40% smaller than Ewwii's binary

meh is **not** a research project. It uses boring technology (GTK4, Rust, yuck)
with good defaults and a focus on performant idle.

-----

## Features inherited from eww

Everything eww can do, meh can do. Notable features:

- **Poll sources** (`defpoll`) — periodic shell command execution, output stored
  as a variable. Subprocess gating: polls only run when at least one window is
  open. On wayland idle compositors, the bar process can sit at 0% CPU while
  the screen is locked / blanked.

- **Listen sources** (`deflisten`) — long-lived subprocess, each stdout line
  updates a variable. Used for event streams (playerctl status changes, DBus
  monitors, inotifywait).

- **Subscribe sources** (`defsubscribe`) — kernel inotify or DBus signal
  listeners.  No polling overhead, instant response to file changes.

- **Reactive bindings** — attribute expressions like `:text {CPU + "%"}` are
  re-evaluated only when their dependencies change.  No full widget tree
  rebuilds.

- **Layer shell window management** — `defwindow` with `:monitor`, `:geometry`,
  `:stacking`, `:focusable`, `:decorations`.

- **GTK4 native widgets** — `box`, `button`, `label`, `image`, `scale`,
  `progress`, `circular-progress`, `scroll`, `overlay`, `revealer`, `stack`,
  `expander`, `checkbox`, `input`, `calendar`, `combo-box-text`, `color-button`,
  `literal`, `systray`, `shader`.

- **CSS styling via SCSS** — `meh.scss` compiled with `grass` (pure Rust SCSS
  compiler). Same CSS widget model as GTK4.

- **Animations** — `AdwTimedAnimation` for interpolated attribute transitions.
  Declarative `:animate-duration`/`:animate-easing` on any widget.

- **Hot reload** — `meh reload` re-parses yuck, rebuilds only changed windows.
  Running state (var values) preserved across reload.

- **Multi-monitor** — windows open per-monitor via `meh open NAME --monitor N`.
  Monitor hotplug detected via GdkDisplay; windows auto-relocate.

- **IPC with `bincode`** — typed IPC (not JSON) between `meh daemon` and
  `meh` CLI.  Protocol: length-prefixed bincode frames over Unix socket.

-----

## Architecture

```
meh/
├── crates/
│   ├── yuck/              # S-expression parser (from elkowar/eww, MIT). No GTK.
│   ├── core/              # Widget tree IR, reactive var graph, IPC types.
│   ├── gtk4-impl/         # GTK4 widget implementations, reactive bindings.
│   ├── layer-shell/       # gtk4-layer-shell window placement.
│   ├── script-vars/       # defpoll / deflisten / defsubscribe.
│   ├── notifier-host/     # StatusNotifierHost. Opt-in systray. From elkowar/eww (MIT).
│   ├── daemon/            # tokio runtime, IPC server, hot reload supervisor.
│   └── cli/               # meh binary — daemon + client subcommands.
├── examples/
│   ├── minimal-bar/      # Minimal yuck-only config.
│   └── widget-test/      # All-widgets verification config.
├── benches/
└── AGENTS.md             # This file.
```

**Crate rules:**
- `yuck/` and `core/` must not depend on `gtk4`.
- `gtk4-impl/` must not depend on `script-vars` directly (event handlers route
  through `daemon/`).

-----

## Architecture decisions (ADRs)

Append-only. Add a new ADR when you make a decision not already covered.
Never edit accepted ADRs in place; supersede with a new one.

### ADR-0001 — GTK4, not GTK3

**Supersedes:** eww's GTK3 choice.
**Status:** Accepted · **Date:** 2026-05-20

**Context.** eww runs on GTK3. The GTK3 Layer Shell implementation has known
bugs, and GTK3 is increasingly deprecated/removed from distributions.  Ubuntu
24.04 (used in CI) ships GTK 4.14.5 from apt.  The Ewwii prototype proved
GTK4 works.

**Decision.** Use the `gtk4-rs` 0.10.x crate with `v4_14` feature flag.
Minimum API version: GTK 4.14.5 (Ubuntu 24.04).  This means no GTK 4.16+
API calls (e.g. `StringList`, `NoInspector`).  The `v4_14` flag sets the
pkg-config requirement; symbols are checked at compile time by gtk4-rs,
so the CI runner enforces the constraint.

**Consequences.** Positive: GTK4 is actively developed, Wayland-native, and
available on all modern distros.  Negative: no GTK4-to-X11 bridge handles
the X11 use case by default (out of scope).  Layer shell protocol (wlr-layer-shell
and ext-layer-shell) works the same on GTK3 and GTK4.

### ADR-0002 — Reactive bindings, not full tree rebuilds

**Supersedes:** eww's approach (same principle, cleaner implementation).
**Status:** Accepted · **Date:** 2026-05-20

**Context.** eww rebuilds the entire widget tree on every variable change.
Ewwii improved this but still rebuilds large subtrees.

**Decision.** meh implements a reactive binding system: each widget attribute
that references a variable is stored as a `Binding` struct with a setter
closure.  On variable change, only the affected bindings are re-evaluated.
No tree traversal, no DOM diff, minimal GTK property set calls.

**Consequences.** Positive: ~10× faster variable updates vs eww for typical
bars.  Negative: the binding system is more code than a simple tree rebuild
(~200 lines vs ~50).  Acceptable tradeoff for the UI responsiveness gain.

### ADR-0003 — Lazy polling (ADR-0008 in earlier docs)

**Status:** Accepted · **Date:** 2026-05-21

**Context.** When no windows are visible (screen off, compositor idle), polling
shell commands every 1-2 seconds wastes CPU and wakes the compositor.

**Decision.** Poll subprocesses are gated on `windows_open: AtomicBool`.
When the last window is closed, all poll timers skip their ticks until a
window is opened again. Listen subprocesses are unaffected (they carry
near-zero idle cost, blocking in kernel read).

**Consequences.** Positive: daemon CPU drops to ~0.1% when no windows visible.
Negative: slight complexity in the poll loop (one `AtomicBool` check per tick).
Listen processes still consume memory (~5 MB RSS each) even when no windows
are visible; users are advised to use `defsubscribe` where possible.

### ADR-0004 — No native plugin system

**Status:** Accepted · **Date:** 2026-05-21

**Context.** eww has no plugin system.  Proposals for dynlib plugins (`.so`)
would require a stable ABI, linker symbol management, and expose GTK FFI
through `dlopen`.

**Decision.** meh will never have native plugins.  Scripting and plugin
systems belong in **meh2**, which extends meh with Rhai and a pure-script
plugin model.  meh remains pure yuck+CSS, minimal and focused.

**Consequences.** Positive: no ABI surface, no security review of plugin
sandboxes, no stability risk.  Negative: users who want dynamic behaviour
must use meh2 instead of meh.  Acceptable: meh2 is the same binary format,
same config format, same IPC protocol — it's an upward-compatible superset.

### ADR-0005 — Rust 2024 edition

**Status:** Accepted · **Date:** 2026-05-21

**Context.** The Rust 2024 edition is stable as of Rust 1.85 (2025-02-20).
It enables `gen` blocks, `unsafe_extern_blocks` by default, and stricter
`unsafe` rules.

**Decision.** Use `edition = "2024"` for the entire workspace.  This is a
deliberate choice to signal that meh targets modern Rust and does not
carry legacy edition baggage.

**Consequences.** Positive: `unsafe` usage is explicit, `gen` blocks available
if needed, future-proof.  Negative: Rust < 1.85 cannot build meh.  Acceptable:
the CI runner uses stable (currently 1.86+).

### ADR-0006 — `jq` and `tz` are opt-in features

**Supersedes:** eww's always-on approach.
**Status:** Accepted · **Date:** 2026-05-21

**Context.** `jq` support links `jaq-core`, `jaq-interpret`, `jaq-parse`, and
`jaq-std` (~4.3 MiB debug, ~1.7 MiB stripped).  `tz` (timezone support in
`formattime`) links `chrono-tz` (~1.0 MiB stripped).  Together they add
~2.7 MiB to the stripped binary.

**Decision.** Feature-gate `jq` and `tz` behind Cargo features.  The `minimal`
profile excludes both.  `default` and `full` include both.

**Consequences.** Positive: `minimal` binary is 4.2 MiB stripped (vs 6.9 MiB
default).  Negative: users relying on `jq` or `tz` and building with `minimal`
must opt in explicitly.

### ADR-0009 — Declarative animations via `AdwTimedAnimation`

**Status:** Accepted · **Date:** 2026-05-22

**Context.** eww has no animation support.  Animations must be implemented
externally (CSS transitions in GTK3 were limited).  GTK4's built-in animation
framework (`GtkTimedAnimation`, `AdwTimedAnimation`) provides interpolation
between two values over a duration with an easing function.

**Decision.** Use `libadwaita::TimedAnimation` for widget attribute animations.
The `animations` Cargo feature (part of `default` profile) gates the
`libadwaita` dependency.  `TimedAnimation` supports: arbitrary `from`/`to`
values via `CallbackAnimationTarget`, configurable easing via
`AdwEasing` variants, and frame-sync with the GTK compositor clock.
Properties that support animation: `opacity` and progress `value`.

**Consequences.** Positive: smooth 60fps animations without manual Cairo/GL
code.  Negative: depends on `libadwaita` (adds ~500 KB).  The `animations`
feature is in `default` profile but excluded from `minimal`.

### ADR-0010 — `defsubscribe` for inotify and DBus

**Status:** Accepted · **Date:** 2026-05-22

**Context.** Polling files every 1-60 seconds is wasteful.  Linux has kernel
inotify for file changes and DBus signals for system events.  Both are
available without external dependencies (inotify via `notify` crate, DBus
via `zbus`).

**Decision.** Add `defsubscribe` as a third script-var source alongside
`defpoll` and `deflisten`.  Two backends:
- **`defsubscribe :file "path"`** — watches a file with inotify.  File contents
  are read and emitted on each change.  Handles missing files (watches parent
  directory for creation).  Uses the `notify` crate.
- **`defsubscribe :dbus-service ...`** — listens to `PropertiesChanged` signals
  on a specific DBus service/object/interface/property.  Uses `zbus` `MessageStream`.
  Supports both system and session buses.

Both backends are opt-in via `inotify-vars` and `dbus-vars` Cargo features,
both in the `default` profile.

**Consequences.** Positive: zero-polling file and DBus monitoring.  Negative:
`inotify-vars` pulls in `notify` (~200 KB); `dbus-vars` pulls in `zbus`
(~1.6 MB).  Both are excluded from `minimal` profile.

-----

## Build profiles and Cargo features

### Profiles

| Profile | Command | Binary size | Includes |
|---|---|---|---|
| `minimal` | `cargo build --release --no-default-features --features minimal` | 4.2 MiB | Core widgets, poll/listen (shell only), no systray |
| `default` | `cargo build --release` | 6.9 MiB | Minimal + systray, subscribe vars, animations, jq, tz |
| `full` | `cargo build --release --features full` | 6.9 MiB | Default + GL shader |

### Features

| Feature | Adds | Idle cost when unused | Profile |
|---|---|---|---|
| `systray` | StatusNotifierHost, `(systray)` widget | Zero — no DBus connection made until first systray widget | `default` |
| `inotify-vars` | `defsubscribe :file` via `notify` crate | Zero — no inotify fd opened until first `:file` var | `default` |
| `dbus-vars` | `defsubscribe :dbus-service` via `zbus` crate | Zero — no DBus connection made until first `:dbus` var | `default` |
| `animations` | `AdwTimedAnimation` widget transitions | Zero — `libadwaita` init deferred until first animated widget | `default` |
| `jq` | `jq()` function in simple expressions | Zero — jaq filter compiled lazily on first call | `default` |
| `tz` | Timezone parameter in `formattime()` | Zero — `chrono-tz` data static, no init cost | `default` |
| `shader` | `(shader)` widget via GtkGLArea | Zero — no GL context created until first shader widget | `full` |
| `builtin-default-config` | Embedded minimal bar; works with no config dir | Zero — only checked when no config file on disk | opt-in |

-----

## Coding conventions

### Rust

- Use the 2024 edition idioms (`use` and `impl Trait` syntax).
- `cargo fmt` and `cargo clippy` must pass before every commit.
- Prefer `anyhow::Result` for fallible functions. Define custom error types
  only when callers need to discriminate error variants.
- `tracing` for logging. `tracing::error!` for daemon-crashing bugs,
  `tracing::warn!` for recoverable issues, `tracing::debug!` for verbose
  tracking.
- Avoid `unwrap()` and `expect()` in library code. Library functions return
  `Result` or handle the None case explicitly. `unwrap()` in binary entry
  points is acceptable with a clear message.
- Use `thiserror` for error enums.
- Re-export public API from `crate::lib.rs`. Internal modules are
  `pub(crate)` or private.

### GTK4

- Widget constructors return `gtk4::Widget` for top-level and `Result<gtk4::Widget>`
  for fallible builds.
- Reactive bindings use the `BINDING_COLLECTOR` thread-local pattern. Never
  push to binding state outside `collect_bindings()`.
- Attribute resolution: `EvalCtx::eval_attr_*` methods.  Never extract
  `AttrEntry` values directly.

### Yuck / config

- meh must parse any valid eww yuck config without errors.
- `defpoll` intervals are in seconds (same as eww).  Sub-second intervals
  are not supported (use `deflisten` for high-frequency updates).
- New yuck attributes default to `:optional true` unless semantically required.

### Performance

- Benchmark before claiming performance improvements.
- `cargo bloat` for binary size investigations.
- Profile with `perf` or `tracy` for runtime performance.
- Prefer `defsubscribe` over `deflisten` over `defpoll`.

-----

## Performance principles

1. **Idle cost is the headline metric.** A clock + workspaces config sits
   at < 0.1% CPU on modern hardware. Anything that moves this needs justification.
1. **Lazy windowing.** A defined-but-closed window allocates nothing until opened.
1. **Variable graph.** Re-render only widgets whose dependencies actually changed.
1. **No re-parse on update.** Yuck compiles to an IR at load; updates mutate state.
1. **Subscribe over poll.** Use DBus signals, inotify, netlink before `poll`.
1. **Benchmark before claiming perf.** `criterion` for hot paths. Baseline in
   `benches/baselines/`.
1. **`cargo bloat` is a tool we use.** Surprising entries get investigated.
1. **Binding updates build minimal var maps.** `Binding::update()` and `LoopBinding::update()`
   previously cloned the entire `global_vars` HashMap (~50 entries, many multi-KB JSON strings)
   on every 33ms tick. Now they collect var refs from the expression and build a HashMap with
   only those vars (typically 1–3 per binding). This eliminated allocation churn that inflated
   RSS from ~48 MB to 60–100 MB.
1. **Binding var_refs cached at build time.** `collect_var_refs()` is called once at binding
   creation. The per-tick `update()` uses the cached refs directly, eliminating the tree walk
   and Vec allocation on every tick.
1. **Bindings skipped when vars not changed.** `update_bindings()` accepts a `changed_vars` set
   and skips bindings whose cached `var_refs` don't intersect. Only bindings referencing the
   actual changed vars are evaluated.
1. **Pending var map cleared when windows closed.** `forward_var_updates()` clears its
   pending HashMap when no windows are open, preventing deflisten sources from
   accumulating stale var values in memory during idle periods.

-----

## Roadmap

**Phase 1 — Foundations** (get to feature parity with eww)

- [x] Fork Ewwii-sh/ewwii. Strip X11 backend.
- [x] Pull `crates/yuck/` from elkowar/eww. Wire into Ewwii's widget backend.
- [x] Reactive binding system per ADR-0007.
- [x] Poll subprocess gating per ADR-0008.
- [x] Implement all eww widgets (box, button, label, image, scale, progress,
      circular-progress, scroll, overlay, revealer, stack, expander, checkbox,
      input, calendar, combo-box-text, color-button).
- [x] `defsubscribe :file` + `defsubscribe :dbus-service` per ADR-0010.
- [x] `(literal)` widget for raw CSS boxes.
- [x] `(launcher)` widget for application launcher popups.
- [x] `(systray)` widget for StatusNotifierItem tray icons.
- [x] `(shader)` widget for GL shader effects.
- [x] `(scroll)` with reactive hover-to-scroll.
- [x] Animations per ADR-0009.
- [x] Hot reload (`meh reload`).
- [x] Multi-monitor support.
- [x] CSS via SCSS (grass crate).
- [x] IPC via bincode Unix socket.
- [x] Three build profiles.
- [x] CI via GitHub Actions.
- [x] `builtin-default-config` feature.
- [x] `meh-default-config/` AUR PKGBUILD variant.

**Phase 2 — Systray interactivity (complete)**

- [x] Systray menu support.
- [x] Systray icon change signal handling.
- [x] (systray) :onleftclick / :onrightclick / :onmiddleclick.

**Phase 3 — Stabilisation (current)**

- [ ] (future) Fix any eww-compatibility issues found in real usage.
- [ ] (future) Performance tuning for reported hot paths.

**meh is not expected to grow beyond Phase 3.** Further feature work moves to meh2.

-----

## Rules for the agent

- **Read this AGENTS.md top-to-bottom at the start of every session.**
- **The prime directive is non-negotiable.** Lighter, faster, simpler than eww.
- **meh2 is the active fork.** Cherry-pick pure-meh fixes from meh into meh2.
  Never merge meh2 into meh.
- **ADRs are append-only.** New ADR for new decisions. Supersede, never edit.
- **No scripting. No plugins.** Those go in meh2. Stop and ask if the task
  implies them.
- **Measure perf before claiming improvement.** CI benchmarks are in
  `benches/baselines/`.
- **When unsure between two approaches, ask.** Don't pick silently.
- **`cargo fmt && cargo clippy` before every commit.**

-----

## Getting started — step by step

1. **Check the upstream clones exist** at `../eww-upstream`, `../eww-gtk4-branch`,
   `../ewwii-upstream`. If any is missing, print the `git clone` command and stop.
   ```
   git clone https://github.com/elkowar/eww          ../eww-upstream
   git clone -b gtk4 https://github.com/elkowar/eww  ../eww-gtk4-branch
   git clone https://github.com/Ewwii-sh/ewwii       ../ewwii-upstream
   ```

1. **Build and run the minimal bar:**
   ```bash
   cd ~/Projects/meh
   cargo build --release --no-default-features --features minimal
   sudo install -m755 target/release/meh /usr/bin/meh
   meh daemon &
   meh open bar
   ```

1. **Customise your config:**
   ```bash
   cp -r ~/Projects/meh/examples/minimal-bar/* ~/.config/meh/
   # edit ~/.config/meh/meh.yuck, meh.scss, mehrc
   meh reload
   ```

-----

## Agent setup

When you start a fresh agent session in this repo:

1. Read `/home/sickhate/Projects/meh/AGENTS.md` top-to-bottom if you haven't
   yet this session.
1. Check `git status` to see the current state.
1. Run `cargo build --release` to confirm the workspace resolves.
1. Start working on the task.

-----

## First-session prompt

The text below is given verbatim to a new agent at the beginning of its
first session. It replaces the need for extensive hand-holding in the
AGENTS.md itself.

```
You are working in the meh repository at /home/sickhate/Projects/meh.

meh is a GTK4 Wayland-only widget system, forked from Ewwii-sh/ewwii. It is the
spiritual successor to elkowar/eww.  Binary: meh.  Config: ~/.config/meh/.

This project has a sibling: ~/Projects/meh2, which extends meh with a Rhai
scripting engine and plugin system.  meh2 cherry-picks bugfixes from meh;
meh never merges from meh2.

Read /home/sickhate/Projects/meh/AGENTS.md top-to-bottom before proceeding.
Follow its instructions.  If a task falls outside meh's scope, suggest meh2
instead.
```
