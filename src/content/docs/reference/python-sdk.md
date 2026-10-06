---
title: Python SDK
description: 零第三方依赖的 ctypes 封装：成员表、命名空间与异常模型。
---

## 安装与导入

无需 `pip install`——`ctypes` 是标准库。把 `examples/python/` 加入路径即可：

```python
from uda import Uda

with Uda() as uda:                                  # 自动解析
    ...
with Uda(library_path="/opt/uda/libuda_ffi.so") as uda:   # 显式指定
    ...
```

库解析顺序：显式 `library_path` 参数 → `UDA_LIBRARY` 环境变量 → `cargo metadata` 报告的 target 目录 → 仓库内常见构建目录 → 系统动态库搜索路径。

## 常量类

| 类 | 成员 |
|----|------|
| `Theme` | `DARK` / `LIGHT` / `UNKNOWN` |
| `FillMode` | `CROP` / `FILL` / `FIT` / `STRETCH` |
| `WakeLockType` | `DISPLAY` / `SYSTEM` |
| `MediaCommand` | `PLAY` / `PAUSE` / `TOGGLE` / `NEXT` / `PREVIOUS` / `STOP` |
| `PlaybackStatus` | `PLAYING` / `PAUSED` / `STOPPED` / `UNKNOWN` |
| `SessionAction` | `LOCK` / `LOGOUT` / `SUSPEND` / `HIBERNATE` / `REBOOT` / `SHUTDOWN` |
| `SessionCapability` | `MANAGEMENT` / `LOCK` / `LOGOUT` / `SUSPEND` / `HIBERNATE` / `REBOOT` / `SHUTDOWN` |

## `Uda`

| 成员 | 类型 | 说明 |
|------|------|------|
| `Uda(library_path=None)` | 构造 | 加载库并声明原型 |
| `uda.theme` | `str` 属性 | `'dark'` / `'light'` / `'unknown'` |
| `uda.accent_color` | `tuple[int,int,int,int] \| None` | RGBA；平台无此概念时为 `None` |
| `uda.wallpaper` | `str \| None` 属性（可写） | 当前壁纸路径；赋值等同 `set_wallpaper(path, FILL)` |
| `uda.set_wallpaper(path, fill_mode)` | 方法 | 显式指定填充模式 |
| `uda.notify(title, body, icon, actions, app_name)` | 方法 → `int` | 返回通知 id |
| `uda.wakelock(lock_type, reason)` | 方法 → `WakeLock` | 上下文管理器 |
| `uda.create_tray_icon(name, tooltip, icon)` | 方法 → `TrayIcon` | |
| `uda.create_tray_menu()` | 方法 → `TrayMenu` | |
| `uda.media` | `_MediaController` 属性 | 媒体命名空间 |
| `uda.session` | `_SessionController` 属性 | 会话命名空间 |
| `uda.release_all()` | 方法 | 释放本对象持有的全部锁与托盘资源 |
| `__enter__` / `__exit__` | 上下文管理 | `with Uda() as uda:` |

## 命名空间 `uda.media`

| 成员 | 类型 |
|------|------|
| `now_playing` | `MediaTrack \| None` 属性 |
| `status` | `str` 属性（`PlaybackStatus` 常量） |
| `send(command)` | 方法，接受指令名字符串或常量 |
| `play()` / `pause()` / `play_pause()` / `next()` / `previous()` / `stop()` | 便捷方法 |

`MediaTrack` 字段：`title`、`artist`（`str`；播放器发布多位艺人时已用 `", "` 连接）、`album`、`duration_ms`、`position_ms`。

## 命名空间 `uda.session`

| 成员 | 类型 |
|------|------|
| `capabilities` | `dict[str, bool]` 属性 |
| `supports(action)` | `bool`，`action` 为动作名 |
| `lock()` / `logout()` / `suspend()` / `hibernate()` / `reboot()` / `shutdown()` | 方法 |

## 通知

```python
uda.notify("下载完成", "report.pdf 已保存到 ~/Downloads")
uda.notify("更新可用", "v0.2.1 已发布",
           icon="/home/me/Pictures/ok.png",
           actions={"open": "查看详情", "later": "稍后提醒"},
           app_name="我的应用")
```

| 参数 | 含义 |
|------|------|
| `title` | 单行标题 |
| `body` | 多行正文，可为空 |
| `icon` | 图标路径或 URI，可为空。请用绝对路径——相对路径按进程工作目录解析 |
| `actions` | `{key: label}` 表，如 `{"open": "查看详情"}` |
| `app_name` | 在 Windows 上即 toast 的 AppUserModelID；留空则使用通用身份 `UniDesktop.Notification` |

Windows 上 toast 的动作*按钮*需要 MSIX 打包的激活器，因此 `actions` 会被接受但只呈现为文本——toast 本身仍正常弹出。

## `WakeLock`

| 成员 | 说明 |
|------|------|
| `handle` | 库分配句柄 |
| `release()` | 释放；重复调用安全 |
| `__enter__` / `__exit__` | `with` 语句 |

## `TrayIcon`

| 成员 | 类型 |
|------|------|
| `handle` | 句柄；销毁后为 0 |
| `tooltip` | `str` 属性（可写）；超 127 字符被截断 |
| `icon` | `str` 属性（可写）；从图片文件设置 |
| `visible` | `bool` 属性（可写） |
| `menu` | `TrayMenu \| None` 属性（可写） |
| `wait()` | 阻塞至 `stop()` / `destroy()` |
| `stop()` | 让 `wait()` 返回，不注销 |
| `destroy()` | 注销并销毁 |
| `__enter__` / `__exit__` | `with` 语句 |

## `TrayMenu` / `TrayItem`

| 成员 | 说明 |
|------|------|
| `add_text(label, callback)` | 文本项，回调签名为 `(item_id, user_data) -> None` |
| `add_checkbox(label, checked, callback)` | 复选框，回调签名为 `(item_id, checked, user_data) -> None`，接收点击后的新状态 |
| `add_separator()` | 分隔线 |
| `handle` | 句柄 |
| `items` | 已添加的行（含分隔线），按顺序 |
| `item_for(item_id)` | 按 id 查找 |
| `destroy()` | 销毁菜单句柄 |

`TrayItem` 字段：`item_id`、`label`、`kind`、`checked`、`enabled`。

图标可以是 `.png` 文件：SDK 读取文件后用内置的纯标准库 PNG 解码器解码并提交 RGBA——Linux 的 `StatusNotifierItem` 把 `Path` 解释为 freedesktop 图标**主题名**，直接传文件路径什么都看不到。图像在提交前会按最长边 32 px 降采样。

回调运行在托盘工作线程上，必须尽快返回，且不能直接操作 UI——请转发到宿主自己的事件循环。

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

## 媒体与会话

```python
track = uda.media.now_playing      # 无播放器运行时为 None
if track:
    print(f"{track.title} - {track.artist} ({track.duration_ms} ms)")
uda.media.send("play" if uda.media.status != "playing" else "pause")

if uda.session.supports("suspend"):
    uda.session.suspend()          # 破坏性动作——先向用户确认
```

没有可播放内容时 `now_playing` 返回 `None` 而不抛异常。六个会话动作中有五个会结束用户会话或停机，只有 `lock()` 适合自动化执行。

## 异常模型

```python
class UdaError(RuntimeError):
    status: int       # 负状态码
    # 消息由 uda_last_error_message() 提供；读取失败时退回状态码描述
```

```python
from uda import UdaError

try:
    uda.set_wallpaper("/nonexistent.png", FillMode.FIT)
except UdaError as exc:
    print(exc.status, exc)      # (-2, 'Feature not supported: ...')
```

`UdaError` 继承 `RuntimeError`，并携带 `status`（负数状态码）。消息文本来自 `uda_last_error_message()`；读取失败时改用状态码描述。

每个状态码的含义见[状态码](/reference/status-codes/)。

`__del__` 兜底只在忘记释放时触发，正常路径请用 `with` 或显式 `destroy()` / `release()`。

## 相关文档

- [Node.js SDK](/reference/nodejs-sdk/)——同样基于 koffi 的绑定
- [C-ABI 参考](/reference/c-abi/)——底层函数表
- [状态码](/reference/status-codes/)——常量定义与触发场景
