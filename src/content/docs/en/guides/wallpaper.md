---
title: Wallpaper
description: Setting and reading the desktop background with fill modes, multi-monitor targeting and dark/light pairing.
---

## Reading the current wallpaper

```python
from uda import Uda

with Uda() as uda:
    path = uda.wallpaper          # property read; may be None

    if path:
        print("current wallpaper:", path)
    else:
        print("not set, or the platform cannot read it")
```

## Setting a wallpaper

```python
from uda import Uda, FillMode

with Uda() as uda:
    uda.set_wallpaper("~/Pictures/mountains.jpg", FillMode.FILL)
    # or by property assignment (defaults to FILL)
    uda.wallpaper = "/usr/share/backgrounds/gnome/adwaita-l.jpg"
```

```javascript
uda.setWallpaper('~/Pictures/a.png', 'fill');
console.log(uda.wallpaper);
```

## Fill modes

| Mode | Behaviour |
|---|---|
| `crop` | scale to fill, preserving the aspect ratio; the overflow is cropped |
| `fill` | scale to fill, ignoring the aspect ratio |
| `fit` | scale to fit inside, preserving the aspect ratio; letterbox if needed |
| `stretch` | stretch to fill, ignoring the aspect ratio |

`crop` and `fit` preserve the aspect ratio; `fill` and `stretch` do not — which one is "correct" depends on the artwork.

## Dark / light pairing

GNOME keeps two values — `picture-uri` and `picture-uri-dark` — so the background can follow the system colour scheme. UDA writes both when the backend supports it, falling back to a single value otherwise.

The pairing is a GNOME-backend behaviour. It is not covered by a dedicated capability bit — `Capability::SET_WALLPAPER` and `Capability::GET_WALLPAPER` are what the wallpaper backend advertises — so do not gate the dark/light pairing on a capability query.

## Multi-monitor

Where the backend exposes per-monitor state (GNOME monitors via the portal, KDE via plasmashell), UDA targets all screens by default. Per-monitor selection is part of the Display Topology work; until then the set value applies to every screen the backend covers.

## Per-desktop backends

The wallpaper backend does **not** use the XDG Desktop Portal: `org.freedesktop.portal.Wallpaper` exists but is not called. It selects an implementation per desktop instead:

| Desktop | Mechanism |
|---|---|
| GNOME 42+ | the `gsettings` CLI, writing `org.gnome.desktop.background` `picture-uri` and `picture-uri-dark` as a pair |
| KDE Plasma 5 / 6 | D-Bus `org.kde.plasmashell` → `/PlasmaShell` → `evaluateScript` |
| Hyprland | the `hyprpaper` CLI, falling back to `swww` |
| Sway | the `swww` CLI |
| Generic X11 | the `feh` CLI, falling back to `nitrogen` |
| Windows | `SystemParametersInfoW(SPI_SETDESKWALLPAPER)` |

When `gsettings` is missing or fails on GNOME, the backend falls through to the CLI chain. The per-desktop probe order and the exact argument vectors are in [wallpaper specifications](https://github.com/UniDesktop/SDK/blob/develop/docs/internals/wallpaper_specs.md).

Reading the wallpaper goes through the same chain in reverse and reports what is *configured*, which may differ from what is currently rendered.

## Errors

| Symptom | Return |
|---|---|
| the file is unreadable | `UDA_ERR_IO` |
| every fallback tier is unavailable | `UdaError::NotSupported`, whose message lists the tools already tried |
| the platform cannot read the value | `uda.wallpaper` returns `None` |

## See also

- [Capability and fallback](/en/guides/capability-and-fallback/) — the tier chain in detail
- [`docs/internals/wallpaper_specs.md`](https://github.com/UniDesktop/SDK/blob/develop/docs/internals/wallpaper_specs.md) — the full protocol mapping
