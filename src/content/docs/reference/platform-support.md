---
title: 平台支持矩阵
description: 功能 × 桌面环境的完整支持情况，以及每一格所用后端。
---

## 总览

| 功能 | Windows 10/11 | GNOME 42+ | KDE Plasma 5/6 | XFCE | Hyprland | Sway | 通用 X11 |
|------|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| 主题检测 | ✅ | ✅ | ✅ | ✅ | ⚠️ | ⚠️ | ⚠️ |
| 强调色 | ✅ | ✅ | ⚠️ | ⚠️ | ❌ | ❌ | ❌ |
| 壁纸 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| 通知 | ✅[^1] | ✅ | ✅ | ✅ | ⚠️ | ⚠️ | ⚠️ |
| 通知图标 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| 通知按钮 | ⚠️[^2] | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| 防休眠锁 | ✅ | ✅ | ✅ | ✅ | ⚠️ | ⚠️ | ⚠️ |
| 系统托盘 | ✅ | ✅ | ✅ | ⚠️ | ✅ | ✅ | ✅ |
| 媒体播控 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| 会话锁屏 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| 会话注销 | ✅ | ✅ | ✅ | ⚠️ | ⚠️ | ⚠️ | ⚠️ |
| 睡眠 / 休眠 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| 重启 / 关机 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |

`✅` 已验证对接真实后端 · `⚠️` 尽力而为，或依赖额外组件 · `❌` 平台无此概念。

[^1]: 商店安装的运行时（商店版 Python/Node.js）会把通知来源显示为宿主包族名，见 `docs/internals/notification_specs.md` §3.1。
[^2]: 未打包宿主中动作按钮降级为只读文本，见 §3.2。

## ⚠️ 的实际含义

表中的符号只是静态提示，**运行时的真实答案**始终通过 `capabilities()` / `support_level()` 获得，不要以本表作为假设依据：

| 条目 | 回退行为 | 上报结果 |
|------|----------|----------|
| Hyprland / Sway / X11 上的主题检测 | 无系统级配色方案；若某个 GTK 应用写入过值则读取 GSettings | 全部探测均无结果时返回 `Theme::Unknown` |
| 平铺 WM / X11 上的通知 | `org.freedesktop.Notifications` 可用，但动作按钮与图片取决于所装守护进程 | 守护进程过旧时按钮不渲染 |
| 平铺 WM / X11 上的防休眠锁 | ScreenSaver 服务缺失时调用直接失败 | 返回 `UdaError`（连接失败）；该模块没有分级上报 |
| XFCE 上的托盘 | SNI 通过 AppIndicator 兼容的 watcher 工作，但没有原生双击事件 | `DoubleClick` 上报 `SupportLevel::None` |

## 在宿主应用中查询

```rust
use uda_core::capability::SupportLevel;
use uda_core::tray::{TrayFeature, TrayManager};

let manager = uda_platform_linux::LinuxTrayManager::new();
let level = manager.support_level(TrayFeature::DoubleClick);

match level {
    SupportLevel::Full => { /* 绘制双击动作 */ }
    SupportLevel::Partial(reason) => { warn!("该后端上双击为降级行为：{reason}") }
    SupportLevel::None => { /* 隐藏 */ }
}
```

C 宿主通过 `uda_session_capabilities` 与各模块的 `capabilities()` 函数获得同样的信息。

## 每格的后端

### 主题检测

| 环境 | 后端 |
|------|------|
| Windows | 注册表 `AppsUseLightTheme` |
| GNOME | Portal `org.freedesktop.appearance` → `gsettings color-scheme` |
| KDE | `kreadconfig6/5` 读 `Colors:Scheme` |
| XFCE | `xfconf-query -c xsettings` |
| Wayland 平铺 WM | ⚠️ 无法可靠判定，返回 `UNKNOWN` |

### 壁纸

壁纸后端不使用 XDG Desktop Portal：`org.freedesktop.portal.Wallpaper` 存在但未被调用。

| 环境 | 实现 |
|------|------|
| GNOME | `gsettings` CLI，成对写入 `picture-uri` 与 `picture-uri-dark` |
| KDE | D-Bus `org.kde.plasmashell` → `/PlasmaShell` → `evaluateScript` |
| Hyprland | CLI `hyprpaper`，缺失时回退 `swww` |
| Sway | CLI `swww` |
| 通用 X11 | CLI `feh`，缺失时回退 `nitrogen` |
| Windows | `SystemParametersInfoW(SPI_SETDESKWALLPAPER)` |

### 通知

| 环境 | 后端 | 图标 | 按钮 |
|------|------|------|------|
| 任意 Linux DE | `org.freedesktop.Notifications` | ✅ | ✅ |
| Windows | WinRT `ToastNotificationManager` | ✅ | ⚠️[^2] |

Wayland 平铺 WM 需要通知守护进程在跑（`mako`、`swaync`、`dunst` 等），否则 `Notify` 调用报 `ServiceUnknown`。

### 系统托盘

| 环境 | 后端 | 备注 |
|------|------|------|
| Linux | `org.kde.StatusNotifierItem` + `com.canonical.dbusmenu` | GNOME 需 AppIndicator 扩展 |
| Windows | `Shell_NotifyIconW`（`NOTIFYICON_VERSION_4`） | 专用工作线程 + 消息泵 |

### 媒体播控

| 环境 | 后端 |
|------|------|
| Linux | MPRIS v2 over session D-Bus |
| Windows | WinRT SMTC |

### 会话与电源

| 动作 | Linux | Windows |
|------|-------|---------|
| 锁屏 | ScreenSaver `Lock` → `loginctl lock-session` | `LockWorkStation()` |
| 注销 | GNOME/KDE D-Bus 方法 | `ExitWindowsEx(EWX_LOGOFF)` |
| 睡眠/休眠 | logind `Suspend`/`Hibernate` | `SetSuspendState` |
| 重启/关机 | logind `Reboot`/`PowerOff` | `ExitWindowsEx` + `SeShutdownPrivilege` |

Linux 侧注销依赖具体 DE 提供 D-Bus 方法；未识别的桌面环境下该动作能力位为假。

## 非 `Full` 的情形

`SupportLevel` 只有三态：`Full`、`Partial(reason)`、`None`。`Partial` 必须携带原因字符串——调用方要把降级原因展示给用户时，无需再猜"为什么"。

| 功能 × 环境 | 级别 | 原因 |
|-------------|------|------|
| 主题检测 × Wayland 平铺 | `Partial` | 无标准查询途径；所有探测手段均无结果 |
| 强调色 × KDE/XFCE | `None` | 无系统级强调色 |
| 通知按钮 × Windows 未打包 | `Partial` | 需 MSIX 注册 COM 激活器 |
| 托盘 × XFCE | `None` | SNI 无双击事件，该特性未被上报 |
| 防休眠 × Wayland 平铺 | `None` | ScreenSaver 服务缺失时调用直接失败，无 CLI 回退 |

## 版本要求

| 环境 | 最低版本 |
|------|----------|
| Windows | 10（toast 功能在 10/11 上完整） |
| GNOME | 42 |
| KDE Plasma | 5 |
| XFCE | 4.14+ |
| Rust | 1.70 |
| Python | 3.9 |
| Node.js | 16 |

## 尚未覆盖（Phase 3）

全局快捷键、带变更监听的多格式剪贴板、音频端点路由与显示器亮度正在开发中，见 [`plans/phase3_plan.md`](https://github.com/UniDesktop/SDK/blob/develop/plans/phase3_plan.md)。

## 相关文档

- [能力与降级](/guides/capability-and-fallback/)——四级降级链的工作原理
- [故障排查](/guides/troubleshooting/)——某项上报 `Unsupported` 时的处置
- [系统通知](/guides/notification/)——Windows 平台的两项限制
