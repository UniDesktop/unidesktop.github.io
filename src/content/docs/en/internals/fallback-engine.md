---
title: Fallback engine
description: How each of the four tiers probes, decides, and what it returns when everything fails.
---

## The four tiers

```text
Tier 1  XDG Desktop Portal
Tier 2  Native DE D-Bus / IPC
Tier 3  CLI tools (probe PATH)
Tier 4  UdaError::NotSupported (typed error)
```

When all three service tiers fail there is **no panic** and no bare `io::Error` — the caller receives a `UdaError::NotSupported` carrying a diagnosis. This is the mechanical form of AGENTS.md Principle 1: never assume a capability, always degrade.

## Tier 1 — XDG Desktop Portal

**Probe**: does `org.freedesktop.portal.Desktop` exist on the session bus, and is the target interface exported?

**Covers**: appearance (`org.freedesktop.portal.Settings`), wallpaper (`org.freedesktop.portal.Wallpaper`), file selection, screen capture, global shortcuts.

**Why first**: the portal is the freedesktop-standard answer, implemented by the desktop itself, consistent across environments, and already handles consent.

**The catch**: not every desktop implements every interface. GNOME 42+ is complete, KDE is partial, Wayland tiling compositors usually have none. Hence *probe*, never assume.

Every portal call runs inside a timeout, because the portal may present a consent dialog and block indefinitely. A timeout becomes a typed error rather than a hung caller.

## Tier 2 — Native DE D-Bus / IPC

**Probe**: read `$XDG_CURRENT_DESKTOP`, map it to a concrete D-Bus service, or check the compositor socket paths (`$HYPRLAND_INSTANCE_SIGNATURE`, `$SWAYSOCK`).

| Desktop | Service / mechanism |
|---|---|
| GNOME | `org.gnome.desktop.background`, `org.gnome.desktop.interface` |
| KDE | `org.kde.plasmashell` → `/PlasmaShell` → `evaluateScript` |
| Hyprland | `$HYPRLAND_INSTANCE_SIGNATURE` socket → `hyprpaper` / `swww` |
| Sway | `$SWAYSOCK` → IPC |
| Any (session) | `org.freedesktop.ScreenSaver`, `org.freedesktop.Notifications` |

**Why it is needed**: native paths cover what the portal does not, and are usually richer — GNOME's dark/light wallpaper pairing, for example, has no portal equivalent.

## Tier 3 — CLI tools

**Probe**: look for an executable on `PATH` in a fixed order.

| Feature | Tool order |
|---|---|
| Wallpaper (X11) | `feh` → `nitrogen` |
| Wallpaper (Wayland) | `swww` → `hyprpaper` |
| Theme (XFCE) | `xfconf-query` |
| Logout (generic) | `loginctl` |

**Why last**: a CLI tool is a process — expensive to start, coarse to interpret (exit code and stderr only), and not necessarily installed. But on a bare X11 environment with no IPC channel it is the only option, which is why the tier exists at all.

## Tier 4 — typed error

The final tier is not a workaround, it is a **contract**: `UdaError::NotSupported("...")` with a message naming the tier chain that was tried. A host can render it verbatim, or match on the type to hide the feature entirely.

## What the caller sees

The tiers are internal. A host observes only:

| Observation | Meaning |
|---|---|
| `Ok(...)` | some tier answered |
| `SupportLevel::Full` | Tier 1 or 2 answered, with no known limitation |
| `SupportLevel::Partial(reason)` | available, but in a degraded form, with the reason attached |
| `UdaError::NotSupported` | all four tiers are exhausted |

## See also

- [Capability and fallback](/en/guides/capability-and-fallback/) — the host-facing view of the same system
- [XDG Desktop Portal](/en/internals/protocols/portal/) — what Tier 1 does and does not cover
