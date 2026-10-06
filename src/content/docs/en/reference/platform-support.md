---
title: Platform support
description: Per-desktop capability matrix, the tier each backend uses, and how to interpret the ⚠️ entries.
---

## Matrix

| Feature | Windows 10/11 | GNOME 42+ | KDE Plasma 5/6 | XFCE | Hyprland | Sway | Generic X11 |
|------|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| Theme detection | ✅ | ✅ | ✅ | ✅ | ⚠️ | ⚠️ | ⚠️ |
| Accent colour | ✅ | ✅ | ⚠️ | ⚠️ | ❌ | ❌ | ❌ |
| Wallpaper | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Notification | ✅[^toast] | ✅ | ✅ | ✅ | ⚠️ | ⚠️ | ⚠️ |
| Notification icon | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Notification actions | ⚠️[^actions] | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Wake lock | ✅ | ✅ | ✅ | ✅ | ⚠️ | ⚠️ | ⚠️ |
| System tray | ✅ | ✅ | ✅ | ⚠️ | ✅ | ✅ | ✅ |
| Media control | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Session lock | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Session logout | ✅ | ✅ | ✅ | ⚠️ | ⚠️ | ⚠️ | ⚠️ |
| Suspend / hibernate | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Reboot / shutdown | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |

`✅` verified against the real backend · `⚠️` best-effort, or needs an extra component · `❌` the platform has no such concept.

[^toast]: A Store-installed runtime shows the notification source as the host's package family name — see `docs/internals/notification_specs.md` §3.1.
[^actions]: In an unpackaged host, action buttons degrade to a read-only text card — see §3.2.

## What ⚠️ means in practice

The symbols are only a static hint. **The real answer at runtime** always comes from `capabilities()` / `support_level()` — never assume from this table:

| Entry | What the fallback does | Reported as |
|------|----------|----------|
| Theme detection on Hyprland / Sway / X11 | No system-wide colour scheme; reads GSettings if a GTK app has written a value | `Theme::Unknown` when every probe comes up empty |
| Notifications on tilers / X11 | `org.freedesktop.Notifications` works, but actions and images depend on the installed daemon | actions render only if the daemon supports them |
| Wake lock on tilers / X11 | The call fails outright when the ScreenSaver service is absent | a `UdaError` (connection failure); this module has no graded report |
| Tray on XFCE | SNI works through the AppIndicator-compatible watcher, but there is no native double-click event | `DoubleClick` reports `SupportLevel::None` |

## Reporting from a host application

```rust
use uda_core::capability::SupportLevel;
use uda_core::tray::{TrayFeature, TrayManager};

let manager = uda_platform_linux::LinuxTrayManager::new();
let level = manager.support_level(TrayFeature::DoubleClick);

match level {
    SupportLevel::Full => { /* draw the double-click action */ }
    SupportLevel::Partial(reason) => { warn!("double click is degraded on this backend: {reason}") }
    SupportLevel::None => { /* hide it */ }
}
```

From a C host the same information is available through `uda_session_capabilities` and the module `capabilities()` functions.

## Backend used per cell

### Theme detection

| Environment | Backend |
|------|------|
| Windows | Registry `AppsUseLightTheme` |
| GNOME | Portal `org.freedesktop.appearance` → `gsettings color-scheme` |
| KDE | `kreadconfig6/5` reads `Colors:Scheme` |
| XFCE | `xfconf-query -c xsettings` |
| Wayland tiling WM | ⚠️ cannot be determined reliably; returns `UNKNOWN` |

### Wallpaper

The wallpaper backend does **not** use the XDG Desktop Portal: `org.freedesktop.portal.Wallpaper` exists but is never called.

| Environment | Mechanism |
|------|------|
| GNOME | the `gsettings` CLI, writing `picture-uri` and `picture-uri-dark` as a pair |
| KDE | D-Bus `org.kde.plasmashell` → `/PlasmaShell` → `evaluateScript` |
| Hyprland | the `hyprpaper` CLI, falling back to `swww` |
| Sway | the `swww` CLI |
| Generic X11 | the `feh` CLI, falling back to `nitrogen` |
| Windows | `SystemParametersInfoW(SPI_SETDESKWALLPAPER)` |

### Notifications

| Environment | Backend | Icon | Actions |
|------|------|------|------|
| Any Linux DE | `org.freedesktop.Notifications` | ✅ | ✅ |
| Windows | WinRT `ToastNotificationManager` | ✅ | ⚠️[^actions] |

A Wayland tiling WM needs a notification daemon running (`mako`, `swaync`, `dunst`, …), otherwise the `Notify` call fails with `ServiceUnknown`.

### System tray

| Environment | Backend | Note |
|------|------|------|
| Linux | `org.kde.StatusNotifierItem` + `com.canonical.dbusmenu` | GNOME needs the AppIndicator extension |
| Windows | `Shell_NotifyIconW` (`NOTIFYICON_VERSION_4`) | dedicated worker thread + message pump |

### Media control

| Environment | Backend |
|------|------|
| Linux | MPRIS v2 over session D-Bus |
| Windows | WinRT SMTC |

### Session and power

| Action | Linux | Windows |
|------|-------|---------|
| Lock | ScreenSaver `Lock` → `loginctl lock-session` | `LockWorkStation()` |
| Logout | GNOME/KDE D-Bus methods | `ExitWindowsEx(EWX_LOGOFF)` |
| Suspend / hibernate | logind `Suspend` / `Hibernate` | `SetSuspendState` |
| Reboot / shutdown | logind `Reboot` / `PowerOff` | `ExitWindowsEx` + `SeShutdownPrivilege` |

The Linux side of logout depends on the DE providing a D-Bus method; on an unrecognised desktop the capability bit for that action is false.

## Cases that are not `Full`

`SupportLevel` has exactly three variants: `Full`, `Partial(reason)` and `None`. `Partial` must carry its reason — a host that wants to explain a degradation to the user should not have to guess why.

| Feature × environment | Level | Reason |
|-------------|------|------|
| Theme detection × Wayland tiling | `Partial` | no standard query path; every probe comes up empty |
| Accent colour × KDE/XFCE | `None` | no system-level accent colour |
| Notification actions × Windows unpackaged | `Partial` | needs an MSIX-registered COM activator |
| Tray × XFCE | `None` | SNI has no double-click event, so the feature is not reported |
| Wake lock × Wayland tiling | `None` | the call fails outright when ScreenSaver is absent; there is no CLI fallback |

## Version requirements

| Environment | Minimum version |
|------|----------|
| Windows | 10 (toasts are complete on 10/11) |
| GNOME | 42 |
| KDE Plasma | 5 |
| XFCE | 4.14+ |
| Rust | 1.70 |
| Python | 3.9 |
| Node.js | 16 |

## Not yet covered (Phase 3)

Global shortcuts, the multi-format clipboard with change listening, audio endpoint routing and display brightness are in progress — see [`plans/phase3_plan.md`](https://github.com/UniDesktop/SDK/blob/develop/plans/phase3_plan.md).

## See also

- [Capability and fallback](/en/guides/capability-and-fallback/)— how the tier chain works
- [Troubleshooting](/en/guides/troubleshooting/) — what to do when a feature reports `NotSupported`
- [Notifications](/en/guides/notification/) — the two Windows limitations
