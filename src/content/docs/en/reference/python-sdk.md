---
title: Python SDK
description: The ctypes binding in examples/python/uda.py — zero dependencies, one entry class.
---

A pure-stdlib binding (`ctypes` + `zlib`) over the [C-ABI](/en/reference/c-abi/). Callers never see `byref`, `c_void_p`, raw pointers or hex status codes: every out-parameter, string allocation and function-pointer lifetime is handled inside the module, and failures raise a `UdaError` carrying the status code and the library's diagnostic message.

## Loading the library

`Uda()` locates `libuda_ffi` in this order:

1. An explicit `library_path` argument.
2. The `UDA_LIBRARY` environment variable.
3. The target directory reported by `cargo metadata` (with the usual `debug` / `release` variants).
4. Common in-repo build directories relative to `examples/python/`.
5. The system dynamic-library search path.

```python
from uda import Uda

with Uda() as uda:                      # automatic resolution
    ...
with Uda(library_path="/opt/uda/libuda_ffi.so") as uda:
    ...
```

## Constants

| Class | Members |
|---|---|
| `Theme` | `DARK` / `LIGHT` / `UNKNOWN` |
| `FillMode` | `CROP` / `FILL` / `FIT` / `STRETCH` |
| `WakeLockType` | `DISPLAY` / `SYSTEM` |
| `MediaCommand` | `PLAY` / `PAUSE` / `TOGGLE` / `NEXT` / `PREVIOUS` / `STOP` |
| `PlaybackStatus` | `PLAYING` / `PAUSED` / `STOPPED` / `UNKNOWN` |
| `SessionAction` | `LOCK` / `LOGOUT` / `SUSPEND` / `HIBERNATE` / `REBOOT` / `SHUTDOWN` |
| `SessionCapability` | `MANAGEMENT` / `LOCK` / `LOGOUT` / `SUSPEND` / `HIBERNATE` / `REBOOT` / `SHUTDOWN` |

## The entry class `Uda`

| Member | Type | Notes |
|---|---|---|
| `Uda(library_path=None)` | constructor | loads the library and declares every prototype |
| `uda.theme` | `str` property | `'dark'` / `'light'` / `'unknown'` |
| `uda.accent_color` | `tuple[int,int,int,int] \| None` | RGBA; `None` when the platform has no such concept |
| `uda.wallpaper` | `str \| None` property (writable) | the current wallpaper path; assignment is equivalent to `set_wallpaper(path, FILL)` |
| `uda.set_wallpaper(path, fill_mode)` | method | set the wallpaper with an explicit fill mode |
| `uda.notify(title, body, icon, actions, app_name)` | method → `int` | returns the notification id |
| `uda.wakelock(lock_type, reason)` | method → `WakeLock` | also a context manager |
| `uda.create_tray_icon(name, tooltip, icon)` | method → `TrayIcon` | |
| `uda.create_tray_menu()` | method → `TrayMenu` | |
| `uda.media` | `_MediaController` property | the media namespace |
| `uda.session` | `_SessionController` property | the session namespace |
| `uda.release_all()` | method | release every wake lock and tray resource the instance holds |
| `__enter__` / `__exit__` | context manager | `with Uda() as uda:` |

`Uda` is a context manager: leaving the `with` block releases every wake lock, tray icon and menu the instance owns. `uda.release_all()` does the same and is safe to call more than once.

## Namespace `uda.media`

| Member | Type |
|---|---|
| `now_playing` | `MediaTrack \| None` property |
| `status` | `str` property (a `PlaybackStatus` constant) |
| `send(command)` | method; accepts a command name or a constant |
| `play()` / `pause()` / `play_pause()` / `next()` / `previous()` / `stop()` | convenience methods |

`MediaTrack` fields: `title`, `artist` (a `str`; multiple artists are already joined with `", "`), `album`, `duration_ms`, `position_ms`.

## Namespace `uda.session`

| Member | Type |
|---|---|
| `capabilities` | `dict[str, bool]` property |
| `supports(action)` | `bool`; `action` is an action name |
| `lock()` / `logout()` / `suspend()` / `hibernate()` / `reboot()` / `shutdown()` | methods |

## Notifications

```python
uda.notify("下载完成", "report.pdf 已保存到 ~/Downloads")
uda.notify("更新可用", "v0.2.1 已发布",
           icon="/home/me/Pictures/ok.png",
           actions={"open": "查看详情", "later": "稍后提醒"},
           app_name="我的应用")
```

| Parameter | Meaning |
|---|---|
| `title` | single-line title |
| `body` | multi-line detail; may be empty |
| `icon` | icon path or URI; may be empty. Use an absolute path — a relative one resolves against the process working directory |
| `actions` | a `{key: label}` map, e.g. `{"open": "查看详情"}` |
| `app_name` | on Windows this is the AppUserModelID; empty selects the generic `UniDesktop.Notification` |

On Windows, action *buttons* need an MSIX-packaged activator, so `actions` is accepted but rendered as text — the toast itself still appears.

## `WakeLock`

| Member | Notes |
|---|---|
| `handle` | the library-allocated handle |
| `release()` | release; safe to call more than once |
| `__enter__` / `__exit__` | `with` statement |

## `TrayIcon`

| Member | Type |
|---|---|
| `handle` | the handle; `0` after destruction |
| `tooltip` | `str` property (writable); truncated above 127 `char`s |
| `icon` | `str` property (writable); set from an image file |
| `visible` | `bool` property (writable) |
| `menu` | `TrayMenu \| None` property (writable) |
| `wait()` | blocks until `stop()` / `destroy()` |
| `stop()` | makes `wait()` return without unregistering |
| `destroy()` | unregisters and destroys |
| `__enter__` / `__exit__` | `with` statement |

## `TrayMenu` / `TrayItem`

| Member | Notes |
|---|---|
| `add_text(label, callback)` | a text row; the callback signature is `(item_id, user_data) -> None` |
| `add_checkbox(label, checked, callback)` | a checkbox row; the callback receives the state *after* the click |
| `add_separator()` | a separator |
| `handle` | the handle |
| `items` | the rows added so far, including separators, in order |
| `item_for(item_id)` | look up a row by id |
| `destroy()` | destroy the menu handle |

`TrayItem` fields: `item_id`, `label`, `kind`, `checked`, `enabled`.

An icon path may be a `.png`: the SDK reads the file, decodes it with the bundled pure-stdlib PNG decoder and submits RGBA, because Linux's `StatusNotifierItem` interprets a `Path` as a freedesktop icon-**theme name** and would show nothing for a file path. The image is downsampled to a 32 px longest edge before being handed over.

Callbacks fire on the tray worker thread, so they must be cheap and must not touch UI state directly — forward into your own loop instead.

```python
def on_toggle(item_id, checked, user_data):
    print(checked)

with Uda() as uda:
    icon = uda.create_tray_icon("My App", tooltip="我的应用正在运行")
    menu = uda.create_tray_menu()
    menu.add_text("设置", lambda item_id, user_data: print("打开设置"))
    menu.add_checkbox("深色模式", checked=False, callback=on_toggle)
    menu.add_separator()
    menu.add_text("退出", lambda item_id, user_data: icon.stop())
    icon.menu = menu
    icon.wait()
    icon.destroy()
```

## Media and session

```python
track = uda.media.now_playing      # None when no player is running
if track:
    print(f"{track.title} - {track.artist} ({track.duration_ms} ms)")
uda.media.send("play" if uda.media.status != "playing" else "pause")

if uda.session.supports("suspend"):
    uda.session.suspend()          # destructive — confirm with the user first
```

`now_playing` returns `None` rather than raising when nothing is playing. Five of the six session actions end the user's session or stop the machine; only `lock()` is safe to automate.

## Error model

```python
from uda import UdaError

try:
    uda.set_wallpaper("/nonexistent.png", FillMode.FIT)
except UdaError as exc:
    print(exc.status, exc)      # (-2, 'Feature not supported: ...')
```

`UdaError` subclasses `RuntimeError` and carries `status` (the negative code). The message comes from `uda_last_error_message()`; if that cannot be read, the status-code description is used instead.

`__del__` is only a fallback for a forgotten release. On the normal path use `with`, or an explicit `destroy()` / `release()`.

See [status codes](/en/reference/status-codes/) for the meaning of every code.

## See also

- [Node.js SDK](/en/reference/nodejs-sdk/) — the same surface over koffi
- [C-ABI reference](/en/reference/c-abi/) — the underlying functions
- [Status codes](/en/reference/status-codes/) — the constants and their triggers
