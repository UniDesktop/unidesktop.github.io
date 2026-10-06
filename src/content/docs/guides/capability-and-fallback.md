---
title: 能力与降级
description: SupportLevel 三态、四级降级链，以及如何在应用里查询能力。
---

## 能力查询的作用

Linux 桌面环境高度碎片化，Wayland 还严格限制部分能力（例如获取窗口全局坐标）。假设功能必然可用的代码，在其他环境中会直接失败。

UDA 的约定是：**每个功能主动上报其能力等级**，应用据此在调用前分流，而不是在调用后处理异常。

## `SupportLevel` 三态

`SupportLevel` 定义在 `uda_core::capability`，含三个变体：

| 变体 | 含义 | 应用应当 |
|------|------|----------|
| `SupportLevel::Full` | 完整支持 | 正常使用 |
| `SupportLevel::Partial(reason)` | 以受限形式提供（如仅支持部分显示器、模式或格式），并**携带降级原因** | 可用，但不要向用户承诺缺失的部分；`reason` 可直接展示给用户 |
| `SupportLevel::None` | 不可用 | 隐藏入口，或给出说明 |

`reason` 通过 `SupportLevel::reason()` 读取，返回 `Option<&str>`：

```rust
if let Some(reason) = level.reason() {
    warn!("该能力已降级：{reason}");
}
```

## Capability 位标志

底层用一组 `bitflags`，每位对应一项能力：

```rust
use uda_core::capability::Capability;

let caps = manager.capabilities()?;

if caps.contains(Capability::SET_WALLPAPER) {
    // 显示"更换壁纸"按钮
}
```

Rust trait 的 `capabilities()` 返回 `Capability`；C-ABI 侧对应具体数字位。会话模块的能力位见[会话与电源生命周期](/guides/session/#能力位表)。

## 四级降级链

在 Linux 上执行任何 OS 桌面动作时，严格遵循：

```text
Tier 1  XDG Desktop Portal      org.freedesktop.portal.* 是否可用？
   ↓ 否
Tier 2  原生 DE IPC             查 $XDG_CURRENT_DESKTOP，调 GNOME/KDE 的
                                D-Bus 方法，或 Hyprland/Sway 的 Unix socket
   ↓ 否
Tier 3  CLI 工具                探测 PATH 上的 swww / hyprpaper / feh /
                                nitrogen / xfconf-query
   ↓ 否
Tier 4  类型化错误              UdaError::NotSupported("...")
```

以壁纸为例的实际链路（该模块不使用 Portal）：

| 桌面 | 实现 |
|------|------|
| GNOME 42+ | `gsettings` CLI，成对写入 `picture-uri` 与 `picture-uri-dark` |
| KDE Plasma | D-Bus `org.kde.plasmashell` → `/PlasmaShell` → `evaluateScript` |
| Hyprland | CLI `hyprpaper`，缺失时回退 `swww` |
| Sway | CLI `swww` |
| 通用 X11 | CLI `feh`，缺失时回退 `nitrogen` |

:::note[降级链的通用规则与该模块的差异]
四级链是 `AGENTS.md` Principle 2 规定的一般规则。壁纸模块是其中的例外：`org.freedesktop.portal.Wallpaper` 存在但未被调用，后端直接从 Tier 2 开始。防休眠同样走 `org.freedesktop.ScreenSaver.Inhibit` 而非 `org.freedesktop.portal.Inhibit`。目前使用 Tier 1 的只有外观检测（`org.freedesktop.portal.Settings`）。
:::

Windows 只有一级：Win32/WinRT API 在受支持版本上总可用，因此没有 portal、没有桌面 IPC、也没有 CLI 回退链。但 Tier 4 仍然适用——例如 Windows 7 上无深色模式检测，代码返回 `UdaError::NotSupported` 而非 panic。

## 在自己的代码里查询

**Python**：会话模块提供 `capabilities()` / `supports()`；其他功能目前通过异常获知。

```python
with Uda() as uda:
    if uda.session.supports("hibernate"):
        show_hibernate_button()
```

**Node.js**：同上，`uda.session.capabilities` / `uda.session.supports('lock')`。

**Rust**：托盘模块提供按特性的细粒度查询：

```rust
use uda_core::capability::SupportLevel;
use uda_core::tray::{TrayFeature, TrayManager};

let manager = uda_platform_linux::LinuxTrayManager::new();
let level = manager.support_level(TrayFeature::Tooltip);

match level {
    SupportLevel::Full => { /* 绘制 tooltip 字段 */ }
    SupportLevel::Partial(reason) => { warn!("该后端上 tooltip 为降级行为：{reason}") }
    SupportLevel::None => { /* 隐藏 */ }
}
```

其他模块同样暴露能力信息：通知后端有 `capabilities()`，会话后端按动作逐位上报，Windows 通知后端另有 `availability()` 用于查询当前进程能否真正弹出 toast。

## 降级优于返回错误

同一功能在不同环境下应给出可用的降级行为，而不是统一的失败：

| 场景 | 降级行为 |
|------|----------|
| Windows 未打包环境带 `actions` | 通知正常显示，按钮不渲染 |
| Windows 无图标 | 用文本模板，卡片不显示图片 |
| Linux 无强调色（KDE/Wayland） | `accent_color` 返回 `None` |
| Linux 上无系统级主题（Wayland 平铺 WM） | 返回 `Theme::Unknown`，而不是猜测 `Light` |
| 通知守护进程缺失 | `Notify` 调用本身报错，能力位仍上报存在 |

:::note[降级必须可观测]
静默丢弃（例如将图标写入日志后丢弃，或因 schema 错误导致操作中心丢弃整条通知）会使调用方认为操作成功，而用户侧没有任何可见结果。此行为属于缺陷，不属于降级。
:::
