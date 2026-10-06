---
title: 故障排查
description: 按模块列出的症状、原因与处置，覆盖 Linux 与 Windows 两侧的常见失败。
---

每次失败都会带上库自身的诊断文本。请在**发起下一次 UDA 调用之前**，通过 `uda_last_error_message()`（C）或 `UdaError.message`（Python）读取它：该槽位是线程本地且按调用覆盖的。

## 快速分诊

| 症状 | 首先检查 |
|------|----------|
| 调用突然返回 `UDA_ERR_NOT_SUPPORTED` | 平台确实不具备该能力——见[平台支持矩阵](/reference/platform-support/) |
| 所有调用都返回 `UDA_ERR_NOT_SUPPORTED` | 当前不在图形会话内（SSH、cron、systemd 单元） |
| 启动即 `UDA_ERR_DETECTION_FAILED` | 环境变量缺失，见下文 |
| 托盘图标从不出现 | 缺少 StatusNotifierWatcher，或传入的是文件路径 |
| toast 从不出现 | 无法解析 AppUserModelID |
| 媒体指令静默无效 | 没有播放器在运行，或播放器禁用了该操作 |

## 环境

### 启动即 `UDA_ERR_DETECTION_FAILED`

环境检测读取 `XDG_CURRENT_DESKTOP`、`XDG_SESSION_TYPE`、`WAYLAND_DISPLAY`、`HYPRLAND_INSTANCE_SIGNATURE` 与 `SWAYSOCK`。在 cron 任务、systemd 单元或纯 SSH shell 中这些变量不存在，此时如实返回「无法判定」。

在图形会话内运行，或显式设置变量：

```bash
XDG_CURRENT_DESKTOP=GNOME XDG_SESSION_TYPE=wayland WAYLAND_DISPLAY=wayland-0 cargo run --example 01_appearance
```

### SSH 下全部为 `NotSupported`

没有会话总线意味着没有 Portal、没有桌面 IPC、也没有通知。唤醒锁与壁纸同样无法工作。这是降级机制按设计工作的结果，不是缺陷。

## 动态库

### 找不到动态库

**症状**：`UdaError: 找不到 UDA 动态库`

**原因**：尚未构建 `uda-ffi`，或 target 目录被重定向后 SDK 未找到。

**处置**：

```bash
cargo build -p uda-ffi
# 仍失败则显式指定
UDA_LIBRARY=$(find . -name 'libuda_ffi.so' | head -1) python3 examples/python/01_appearance.py
```

SDK 的解析顺序：显式 `library_path` → `UDA_LIBRARY` 环境变量 → `cargo metadata` 报告的 target 目录（含 debug/release 与跨编译目标子目录）。

## 壁纸

### `Feature not supported: Neither feh nor nitrogen`

**症状**：`UdaError::NotSupported`，消息列出全部 CLI 工具名。

**原因**：Tier 1（Portal）、Tier 2（GNOME/KDE/Hyprland/Sway IPC）都不可用，Tier 3 探测 `PATH` 也没找到任何工具。

**处置**：安装任一工具，或换到有 Portal 的桌面环境：

```bash
sudo apt install feh        # Debian/Ubuntu
sudo apt install nitrogen   # 备选
```

查询当前环境能力：

```rust
let caps = manager.capabilities()?;
caps.contains(Capability::SET_WALLPAPER)
```

## 通知

### `ServiceUnknown: org.freedesktop.Notifications`

**症状**：`UdaError`，D-Bus 错误名 `org.freedesktop.DBus.Error.ServiceUnknown`。

**原因**：会话里没有通知守护进程（极简 WM、容器、纯 SSH）。

**处置**：安装并启动一个守护进程：

```bash
sudo apt install notification-daemon   # 或 dunst / mako / swaync
```

能力位 `SEND_NOTIFICATION` 只表示「代码路径存在」，不预判守护进程是否在跑——这是运行时错误。

### Windows 上无 toast

普通的 Win32 进程没有包身份，无参的 `CreateToastNotifier()` 会以 `ELEMENT_NOT_FOUND` 失败，什么都不显示。UDA 的应对是 `CreateToastNotifierWithId(app_name)` 加上 `SetCurrentProcessExplicitAppUserModelID(app_name)`——**两者都必须**，因为 shell 还会把该 id 与进程已注册的 AUMID 比对。

若仍失败，显式传入 `app_name`；留空则使用 `UniDesktop.Notification`，这没有问题，但已注册自有 AUMID 的宿主会保留原有值。

直接查询结果：

```rust
let setting = WindowsNotificationManager::new().availability()?;
// DisabledForApplication → 系统设置中该应用的通知被关闭
```

### toast 来源显示为包族名

通过 Microsoft Store 安装的运行时中，shell 已把进程绑定到包身份，该身份会覆盖 AppUserModelID。UDA 无法覆盖它，卡片顶部的来源行显示的是宿主包名。

### Windows 上动作按钮不显示

toast 按钮需要一个只有打包应用才能注册的激活处理器。在未打包宿主中，`actions` 因 trait 对齐而被接受，但卡片降级为只读文本。通知本身仍会显示。

### 通知中没有图标

`app_icon` 是填入 toast 模板 `<image>` 节点的**调用方位图**，与 Windows 从 AUMID 派生的身份图标是两回事。裸路径会被规范化为 `file://` URI，因为 toast 平台从 shell 自身的上下文解析 `src`，而不是发送方的工作目录。库只校验语法——文件不存在时只会产出无图卡片。

## 系统托盘

### Linux 上图标从不显示

D-Bus 会话总线必须可达，且需有 StatusNotifierWatcher 占有 `org.kde.StatusNotifierWatcher`。

```bash
# watcher 是否已注册？
busctl --user list | grep -i statusnotifierwatcher

# UDA 的 item 是否已导出？
busctl --user tree org.kde.StatusNotifierItem-<app>-<pid>-0 | head
```

GNOME 上缺少 watcher 时安装 AppIndicator 扩展——GNOME 默认没有托盘区域。在裸窗口管理器上运行 `mate-panel` 或任何 SNI 兼容的托盘。

### 调用 `uda_tray_set_icon_path` 后图标空白

这在 Linux 上是预期行为：该值被当作 **freedesktop 图标主题名**而非路径，桌面环境去主题目录查找自然找不到。

改用 SDK 的类型化 API，它解码并提交像素：

```python
icon.icon = "icons/UniDesktop_3D_transparent_mini.png"   # 像素路径
```

### 图标红蓝通道互换

自定义转码路径中的字节序缺陷。`IconPixmap` 的「ARGB32」指的是**字节序 B, G, R, A**，且行序自下而上。按 A, R, G, B 写入会互换红蓝。UDA 的后端已处理该问题——这只影响手写转码的路径。

### 托盘调用返回 `UDA_ERR_INVALID_ARGUMENT`

查找按不透明句柄进行：`0` 表示「无句柄」，句柄一次性有效，图标表与菜单表相互独立。常见原因是复用已销毁的句柄，或把菜单句柄传给了图标函数。

### 托盘回调不触发 / 点击无反应

**原因**：Windows 侧回调在**工作线程**上执行，若在回调里做阻塞操作会卡住消息泵；Linux 侧同理。

**处置**：回调里只做最少工作，把实际逻辑 post 到主线程。详见[系统托盘](/guides/tray/#线程模型)。

## 媒体

### 媒体控制：now_playing 一直为 None

**原因**：没有播放器在运行，或播放器未实现 MPRIS v2（部分浏览器、旧版客户端）。

**处置**：用已支持 MPRIS 的播放器验证（`mpv`、`vlc`、Rhythmbox、大多数现代播放器）：

```bash
# 确认播放器已导出 MPRIS 接口
busctl --user list | grep mpris
```

`None` 也可能是播放器未响应的结果——这是正常返回，不是失败。只有平台完全没有媒体后端时才返回 `NotSupported`。

播放器拒绝某条指令（对已暂停的流执行 pause、应用禁用 `Next`）时调用**仍然成功**：API 无法区分「被拒绝」与「已执行」，将这种情况报为错误会让一次正常的切换看起来像失败。

## 会话与电源

### 会话动作返回 NotSupported

**原因**：polkit 未授权，或系统关闭了对应功能（例如无 swap 时无法休眠）。

**处置**：先查能力位；失败时读取错误消息中的 D-Bus 错误名（`AccessDenied` / `NotAuthorized` / `InteractiveAuthorizationRequired`），据此提示用户授权。详见[会话与电源生命周期](/guides/session/#常见失败)。

能力位表示「代码路径存在」，不表示「机器已为该动作配置好」。关闭了休眠的机器仍会上报对应能力位，尝试时才会返回类型化错误。绘制菜单项前先查询 `capabilities()`，并始终向用户确认。

### Windows 上重启或关机因权限失败

账户需要 `SeShutdownPrivilege`，这要求提升权限的进程或本地管理员。Linux 上的对应检查是 polkit 必须允许该会话调用此动作。

### 破坏性动作未经确认即执行

只有 `lock()` 可安全自动化。其余五个动作不可逆；库提供的是目的地，不是防线——确认必须由宿主应用完成。

## 强调色返回 None

**原因**：KDE、XFCE 与 Wayland 平铺 WM 没有系统级强调色概念。

**处置**：这是正常返回值。准备回退色板：

```python
accent = uda.accent_color or (0x33, 0x99, 0xFF, 0xFF)
```

## 图标颜色通道互换（历史缺陷，已修复）

**症状**：托盘图标的红蓝通道互换（蓝色显示为红色）。

**原因**：早期 `IconPixmap` 实现按 A,R,G,B 而非规范要求的 B,G,R,A 排列字节。已在 v0.2.0 修复，见 `docs/internals/tray_specs.md`。

## 测试命令

```bash
cargo test --workspace                       # 全部单元测试
./scripts/test-linux-mock.sh                 # D-Bus mock 夹具测试
cargo check -p uda-platform-windows \
  --target x86_64-pc-windows-gnu --all-targets   # 交叉编译校验
```

D-Bus 相关测试运行在 `dbus-run-session` + `python3-dbusmock` 夹具中，不需要真实桌面环境。

## 诊断 CLI

`crates/uda-cli` 会遍历各子系统并打印结果与应答的层级，是查看当前会话实际支持情况的最快方式：

```bash
cargo run -p uda-cli
```

## 提交缺陷报告

按以下顺序提供：

1. 确切的命令或调用，以及返回的状态码。
2. `uda_last_error_message()` / `UdaError.message` 给出的诊断消息。
3. `XDG_CURRENT_DESKTOP`、`XDG_SESSION_TYPE`、桌面环境及其版本，以及当前是 Wayland 还是 X11。
4. 若日志可见，给出最终应答的降级层级（每次后端交接都会写入 `log::debug!`）。

## 相关文档

- [平台支持矩阵](/reference/platform-support/)——各桌面环境的覆盖情况
- [能力与降级](/guides/capability-and-fallback/)——能力查询与四级降级链
- [会话与电源生命周期](/guides/session/)——会话能力位与平台实现
- [系统托盘](/guides/tray/)——线程模型与生命周期
