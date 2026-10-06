---
title: Capability and fallback
description: The `SupportLevel` tri-state, the four-tier fallback chain, and how to query capabilities from a host.
---

## Capability querying

Linux desktops are heavily fragmented, and Wayland strictly restricts some capabilities outright. Code that assumes a feature exists fails on the next machine.

UDA's contract: **every feature reports its support level**, so the application branches *before* calling.

## `SupportLevel` tri-state

`SupportLevel` is defined in `uda_core::capability` with three variants:

| Variant | Meaning | What your app should do |
|---|---|---|
| `SupportLevel::Full` | fully supported | use it normally |
| `SupportLevel::Partial(reason)` | available in a degraded form (a limited set of monitors, modes or formats), **with the reason attached** | use it, but do not promise the missing part to the user; `reason` can be shown to the user verbatim |
| `SupportLevel::None` | not available | hide the entry, or explain why |

The reason is read through `SupportLevel::reason()`, which returns `Option<&str>`:

```rust
if let Some(reason) = level.reason() {
    warn!("this capability is degraded: {reason}");
}
```

## Capability bit flags

Underneath, capabilities are a set of `bitflags`, one bit per feature:

```rust
use uda_core::capability::Capability;

let caps = manager.capabilities()?;

if caps.contains(Capability::SET_WALLPAPER) {
    // show the "change wallpaper" button
}
```

A Rust trait's `capabilities()` returns a `Capability`; the C-ABI exposes the same information as plain integer bits (see [session capability bits](/en/guides/session/#capability-bits) for that module's table).

## The four-tier chain

Every OS desktop action on Linux strictly follows:

```text
Tier 1  XDG Desktop Portal       is org.freedesktop.portal.* available?
   ↓ no
Tier 2  Native DE IPC            read $XDG_CURRENT_DESKTOP, call GNOME/KDE
                                 D-Bus methods, or Hyprland/Sway sockets
   ↓ no
Tier 3  CLI tools                probe PATH for swww / hyprpaper / feh /
                                 nitrogen / xfconf-query
   ↓ no
Tier 4  Typed error              UdaError::NotSupported("...")
```

When all three service tiers fail there is **no panic** and no bare `io::Error` — the caller gets a `UdaError::NotSupported` carrying a diagnosis.

The actual chain for wallpaper, which does not use the portal:

| Desktop | Mechanism |
|---|---|
| GNOME 42+ | the `gsettings` CLI, writing `picture-uri` and `picture-uri-dark` as a pair |
| KDE Plasma | D-Bus `org.kde.plasmashell` → `/PlasmaShell` → `evaluateScript` |
| Hyprland | the `hyprpaper` CLI, falling back to `swww` |
| Sway | the `swww` CLI |
| Generic X11 | the `feh` CLI, falling back to `nitrogen` |

:::note[The general rule and this module's exception]
The four tiers are the general rule from AGENTS.md Principle 2. Wallpaper is an exception to it: `org.freedesktop.portal.Wallpaper` exists but is not called, so that backend starts at Tier 2. Wake locks likewise use `org.freedesktop.ScreenSaver.Inhibit` rather than `org.freedesktop.portal.Inhibit`. Appearance detection is the only module that currently engages Tier 1, through `org.freedesktop.portal.Settings`.
:::

## Querying from a host application

```rust
use uda_core::capability::SupportLevel;
use uda_core::tray::{TrayFeature, TrayManager};

let manager = uda_platform_linux::LinuxTrayManager::new();

// Cheap, always answers.
let level = manager.support_level(TrayFeature::Tooltip);

match level {
    SupportLevel::Full => { /* draw the tooltip field */ }
    SupportLevel::Partial(reason) => { warn!("tooltip is degraded on this backend: {reason}") }
    SupportLevel::None => { /* hide it */ }
}
```

`support_level` answers from the capability set the backend published at registration, so the value always describes *this* icon's environment. Before a backend publishes, the answer stays `SupportLevel::None` — the honest default when no tray mechanism could be reached.

For features that are not tray-related, the same information is exposed per module: the notification backend has `capabilities()`, the session backend reports one bit per action, and the Python / Node.js SDKs surface the session bits directly.

## Rule: `Partial` is reserved

A backend publishes `Partial` only when it can genuinely deliver a degraded behaviour — Linux double-click synthesis, or a tooltip longer than the soft cap. An absent flag is a plain `None`, never a guess. This keeps "advertised" equivalent to "deliverable".

Because `Partial` carries its reason, the degradation must be describable in one sentence. A backend that cannot say why the feature is degraded does not publish `Partial` — it either delivers the feature or reports `None`.

## See also

- [Fallback engine](/en/internals/fallback-engine/) — how each tier probes and decides
- [Platform support](/en/reference/platform-support/) — the per-desktop matrix
- [Troubleshooting](/en/guides/troubleshooting/) — what to do when something reports `Unsupported`
