---
title: 降级引擎
description: 四级降级链的探测手段、判定条件与设计理由。
---

## 四级链

```text
Tier 1  XDG Desktop Portal
Tier 2  原生 DE D-Bus / IPC
Tier 3  CLI 工具（PATH 探测）
Tier 4  UdaError::NotSupported（类型化错误）
```

三级都失败时**不** panic、不返回裸 `io::Error`，而是返回带诊断信息的 `UdaError::NotSupported`。

## Tier 1：XDG Desktop Portal

**探测**：会话总线上是否存在 `org.freedesktop.portal.Desktop`，且目标接口已导出。

**适用**：外观（`org.freedesktop.portal.Settings`）、壁纸（`org.freedesktop.portal.Wallpaper`）、文件选择、屏幕捕获、全局快捷键。

**优先级理由**：Portal 是 freedesktop 的标准接口，由桌面环境自行实现，行为一致且已内置授权处理。

**覆盖限制**：并非所有桌面都实现全部接口。GNOME 42+ 完整，KDE 部分，Wayland 平铺 WM 通常没有。因此必须探测，不能假设接口存在。

## Tier 2：原生 DE D-Bus / IPC

**探测**：读 `$XDG_CURRENT_DESKTOP`，映射到具体的 D-Bus 服务；或检查 Hyprland/Sway 的 socket 路径（`$HYPRLAND_INSTANCE_SIGNATURE` / `$SWAYSOCK`）。

**适用**：

| 桌面 | 服务 / 手段 |
|------|-------------|
| GNOME | `org.gnome.desktop.background`、`org.gnome.desktop.interface` |
| KDE | `org.kde.plasmashell` → `/PlasmaShell` → `evaluateScript` |
| Hyprland | `$HYPRLAND_INSTANCE_SIGNATURE` socket → `hyprpaper` / `swww` |
| Sway | `$SWAYSOCK` → IPC |
| 通用（会话） | `org.freedesktop.ScreenSaver`、`org.freedesktop.Notifications` |

**作用**：Portal 未覆盖或未实现时的原生路径，功能通常更完整（例如 GNOME 的深浅色壁纸配对）。

## Tier 3：CLI 工具

**探测**：按固定顺序在 `PATH` 上查找可执行文件。

| 功能 | 工具顺序 |
|------|----------|
| 壁纸（X11） | `feh` → `nitrogen` |
| 壁纸（Wayland） | `swww` → `hyprpaper` |
| 主题（XFCE） | `xfconf-query` |
| 注销（通用） | `loginctl` |

**置于末位的原因**：CLI 工具是独立进程，启动开销大、错误处理粗糙（仅能读取退出码与 stderr），且不保证安装。但在没有任何 IPC 通道的裸 X11 环境中，它是唯一可用选项。

**实现要求**：收集 stdout、stderr 与退出码，并在失败时写入错误消息。仅包含 "command failed" 的错误信息对调用方没有诊断价值。

## Tier 4：类型化错误

```rust
UdaError::NotSupported(format!(
    "no wallpaper backend: no Portal, no GNOME/KDE/IPC, and none of [feh, nitrogen] on PATH"
))
```

错误消息列出**已经试过什么**，让调用方能判断是该装工具还是该换环境。

## 降级优于返回错误

同一个 API 在不同环境下应有不同表现：

| 场景 | 降级行为 |
|------|----------|
| Windows 未打包 + 带 actions | 通知显示，按钮不渲染 |
| Windows 无 app_icon | 用文本模板，无图片 |
| Linux 无系统强调色 | 返回 `None` |
| 关闭了休眠的机器 | 能力位为假，运行时 `NotSupported` |

两种行为对照：

- 能力上报支持但调用失败：调用方无法预先分流；
- 能力如实上报，失败时给出可操作原因：调用方可据此绘制菜单或提示。

## 可观测性要求

:::danger[静默丢弃属于缺陷]
Windows 上将图标写入日志后丢弃、而通知仍返回成功的行为，会使调用方认为图片已显示，用户侧却无任何可见结果。v0.2.0 已修正该问题：`app_icon` 要么写入 toast 模板的 `<image>` 节点，要么明确降级为无图模板，两种结果均可从行为上区分。

同理，若生成的 toast 文档因格式错误被操作中心整体丢弃（`Show` 返回成功但无通知显示），同样属于必须修复的实现缺陷，而不是降级。
:::

## Windows 的单级结构

Windows 没有这条四级链：Win32/WinRT API 在受支持版本上总可用，所以没有 Portal、没有桌面 IPC、也没有 CLI 回退。

但 **Tier 4 仍然适用**。例如 Windows 7 上没有深色模式注册表项，代码返回 `UdaError::NotSupported` 而不是假装成功。 tier 的数量取决于平台碎片化程度，而"最终必须给类型化错误"这条不变。
