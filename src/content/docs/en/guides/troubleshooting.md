---
title: Troubleshooting
description: Symptoms, causes and fixes for every failure mode a host is likely to hit.
---

Every failure carries the library's own diagnostic text. Read it with `uda_last_error_message()` (C) or from the raised `UdaError.message` (Python) **before** issuing the next UDA call, because the slot is thread-local and per-call.

## Fast triage

| Symptom | First place to look |
|---|---|
| A call returns `UDA_ERR_NOT_SUPPORTED` out of the blue | the platform genuinely cannot do it — see [platform support](/en/reference/platform-support/) |
| Everything returns `UDA_ERR_NOT_SUPPORTED` | you are outside a graphical session (SSH, cron, systemd unit) |
| `UDA_ERR_DETECTION_FAILED` on startup | environment variables are missing; see below |
| The tray never appears | the StatusNotifierWatcher is missing, or you passed a file path |
| The toast never appears | AppUserModelID could not be resolved |
| A media command silently does nothing | no player is running, or the player disabled that action |

## Environment

### `UDA_ERR_DETECTION_FAILED` immediately

Detection reads `XDG_CURRENT_DESKTOP`, `XDG_SESSION_TYPE`, `WAYLAND_DISPLAY`, `HYPRLAND_INSTANCE_SIGNATURE` and `SWAYSOCK`. In a cron job, systemd unit or bare SSH shell these are absent, and the honest answer is "cannot tell".

Run inside a graphical session, or set the variables explicitly:

```bash
XDG_CURRENT_DESKTOP=GNOME XDG_SESSION_TYPE=wayland WAYLAND_DISPLAY=wayland-0 cargo run --example 01_appearance
```

### Everything is `NotSupported` under SSH

No session bus means no portal, no DE IPC and no notifications. Wake locks and wallpaper cannot work either. This is degradation working as designed, not a bug.

## Shared library

### The shared library cannot be found

**Symptom**: `UdaError: cannot find the UDA shared library`

**Cause**: `uda-ffi` has not been built, or the target directory was redirected and the SDK does not look there.

**Fix**:

```bash
cargo build -p uda-ffi
# if that still fails, point at the artefact explicitly
UDA_LIBRARY=$(find . -name 'libuda_ffi.so' | head -1) python3 examples/python/01_appearance.py
```

The SDK resolves in this order: an explicit `library_path` → the `UDA_LIBRARY` environment variable → the target directory reported by `cargo metadata` (including the debug/release and cross-compilation target subdirectories).

## Wallpaper

### `Feature not supported: Neither feh nor nitrogen`

**Symptom**: `UdaError::NotSupported` whose message lists every CLI tool name.

**Cause**: Tier 1 (the portal) and Tier 2 (GNOME/KDE/Hyprland/Sway IPC) are unavailable, and the Tier 3 `PATH` probe found none of the tools either.

**Fix**: install any one of them, or move to a desktop that provides a portal:

```bash
sudo apt install feh        # Debian/Ubuntu
sudo apt install nitrogen   # alternative
```

Query what the current environment can do:

```rust
let caps = manager.capabilities()?;
caps.contains(Capability::SET_WALLPAPER)
```

## System tray

### The icon never shows on Linux

The D-Bus session bus must be reachable and a StatusNotifierWatcher must own `org.kde.StatusNotifierWatcher`.

```bash
# Is the watcher registered?
busctl --user list | grep -i statusnotifierwatcher

# Is UDA's item exported?
busctl --user tree org.kde.StatusNotifierItem-<app>-<pid>-0 | head
```

If the watcher is missing on GNOME, install the AppIndicator extension — GNOME has no tray area by default. On a bare window manager, run `mate-panel` or any SNI-compatible tray.

### The icon is invisible or blank after `uda_tray_set_icon_path`

That is expected on Linux: the value is treated as a **freedesktop icon-theme name**, not a path. The desktop searches the theme directories and finds nothing.

Use the SDK's typed API instead, which decodes and submits pixels:

```python
icon.icon = "icons/UniDesktop_3D_transparent_mini.png"   # pixel route
```

### The icon has swapped red and blue channels

A byte-order bug in a custom transcoder. `IconPixmap`'s "ARGB32" means **byte order B, G, R, A**, and rows are bottom-up. Writing A, R, G, B instead swaps red and blue. UDA's backend handles this — this only affects hand-rolled transcode paths.

### `UDA_ERR_INVALID_ARGUMENT` from a tray call

Looks up are by opaque handle: `0` means "no handle", a handle is single-use, and the icon and menu tables are separate. Common causes are reusing a destroyed handle, or passing a menu handle where an icon handle is expected.

## Notifications

### No toast on Windows

A classic Win32 process has no package identity, so the parameterless `CreateToastNotifier()` fails with `ELEMENT_NOT_FOUND` and nothing appears. UDA works around this with `CreateToastNotifierWithId(app_name)` plus `SetCurrentProcessExplicitAppUserModelID(app_name)` — **both** are required, because the shell also matches the id against the process's registered AUMID.

If it still fails, pass an explicit `app_name`; leaving it empty selects `UniDesktop.Notification`, which is fine, but a host that already registered its own AUMID keeps it.

Check the answer directly:

```rust
let setting = WindowsNotificationManager::new().availability()?;
// DisabledForApplication → toasts are off for this app in Settings
```

### The toast source shows a package family name

On a Microsoft Store-installed runtime the shell has already bound the process to a package identity, which overrides the AppUserModelID. UDA cannot override that; the card's source line shows the host package.

### Action buttons do not appear on Windows

Toast buttons need an activation handler only a packaged app can register. In an unpackaged host, `actions` is accepted for trait parity but the card degrades to read-only text. The notification itself still displays.

### The icon does not appear in the notification

`app_icon` is the *caller's* bitmap in the toast's `<image>` node, which is separate from the identity icon Windows derives from the AUMID. A bare path is normalised into a `file://` URI, because the toast platform resolves `src` from the shell's context rather than the sender's working directory. Only the syntax is checked — a missing file simply produces a card without a picture.

## Media

### `now_playing` is always `None`

`None` is the normal answer when nothing is playing *or* when no player responds — it is not a failure. Only a platform with no media backend at all returns `NotSupported`.

A player that refuses a command (pausing an already-paused stream, `Next` when the app disables it) is still a **successful** call: the API cannot distinguish "declined" from "done", and reporting an error would make a normal toggle look like a failure.

## Session & power

### `UDA_ERR_NOT_SUPPORTED` from `suspend()` / `hibernate()`

The capability bit means "the code path exists", not "the machine is configured for it". A machine with hibernation switched off still reports the corresponding bit; the attempt then fails with a typed error. Query `capabilities()` before drawing the menu entry — and confirm with the user regardless.

### Reboot or shutdown fails with a privilege error

On Windows the account needs `SeShutdownPrivilege`, which requires an elevated process or a local administrator. On Linux the corresponding polkit check must allow the action for the calling session.

### A destructive action ran without confirmation

Only `lock()` is safe to automate. The other five are irreversible; the library provides the destination, not the guard — the host application must confirm.

## Diagnostics CLI

`crates/uda-cli` walks every subsystem and prints the outcome plus the tier that answered. It is the fastest way to see what the current session actually supports:

```bash
cargo run -p uda-cli
```

## The accent colour returns `None`

**Cause**: KDE, XFCE and Wayland tilers have no system-wide accent-colour concept.

**Fix**: this is a normal return value. Provide a fallback palette:

```python
accent = uda.accent_color or (0x33, 0x99, 0xFF, 0xFF)
```

## Icon channels are swapped (historical defect, fixed)

**Symptom**: the tray icon's red and blue channels are swapped (blue renders as red).

**Cause**: an early `IconPixmap` implementation arranged the bytes as A,R,G,B instead of the required B,G,R,A. Fixed in v0.2.0; see `docs/internals/tray_specs.md`.

## Test commands

```bash
cargo test --workspace                       # all unit tests
./scripts/test-linux-mock.sh                 # D-Bus mock fixture tests
cargo check -p uda-platform-windows \
  --target x86_64-pc-windows-gnu --all-targets   # cross-compilation check
```

D-Bus tests run inside a `dbus-run-session` with `python3-dbusmock` fixtures and need no real desktop environment.

## Diagnostics CLI

`crates/uda-cli` walks every subsystem and prints the outcome plus the tier that answered. It is the fastest way to see what the current session actually supports:

```bash
cargo run -p uda-cli
```

## Reporting a bug

Include, in this order:

1. The exact command or call, and its status code.
2. The diagnostic message from `uda_last_error_message()` / `UdaError.message`.
3. `XDG_CURRENT_DESKTOP`, `XDG_SESSION_TYPE`, the desktop and version, and whether you are on Wayland or X11.
4. The tier that answered, if the log line is visible (`log::debug!` is enabled for every backend handoff).

## See also

- [Platform support](/en/reference/platform-support/) — per-desktop coverage
- [Capability and fallback](/en/guides/capability-and-fallback/) — querying capabilities and the tier chain
- [Session & power lifecycle](/en/guides/session/) — session capability bits and per-platform backends
- [System tray](/en/guides/tray/) — the threading model and lifecycle
