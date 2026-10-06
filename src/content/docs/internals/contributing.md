---
title: 参与贡献
description: 三条核心原则、测试命令与编码约定（人类贡献者视角）。
---

## 三条核心原则

### 1. 永不 panic（Never Panic）

Linux 桌面高度碎片化，Wayland 还严格限制部分能力。任何"假设成功"的代码都会在另一台机器上崩掉。

**规则**：

- 系统调用、D-Bus 调用、环境变量读取上**禁止** `unwrap()` / `expect()`；
- 每个高层功能必须上报能力（`SupportLevel::Full` / `Partial(reason)` / `None`）；
- 新功能失败时先尝试已知替代方案，最后才返回错误。

### 2. 级联降级

Linux 上执行任何 OS 桌面动作：

```text
Portal → 原生 DE IPC → CLI 工具 → UdaError::NotSupported
```

Windows 只有一级（Win32/WinRT 总可用），但最终的类型化错误仍然适用。

### 3. 零重量级依赖

- 不引入 Qt、GTK 或任何 GUI 工具箱；
- Linux 侧用纯 Rust `zbus`；
- Windows 侧用官方 `windows-rs`，且 `uda-platform-windows` 必须保持 `#![cfg(windows)]` 门控，让 Linux 宿主上的 `cargo check --workspace` 永不被打断。

## 编码约定

**错误**：库里用 `thiserror`：

```rust
#[derive(thiserror::Error, Debug)]
pub enum UdaError {
    #[error("Feature not supported: {0}")]
    NotSupported(String),
    #[error("Detection failed: {0}")]
    DetectionFailed(String),
    #[error("Command failed: {0}")]
    CommandFailed(String),
    #[error("IO error: {0}")]
    Io(#[from] std::io::Error),
    #[error("Internal error: {0}")]
    Internal(String),
}
```

**异步**：需要异步处（D-Bus 信号、注册表事件）用 `tokio`，并在可行时提供同步包装。

**日志**：用 `log` crate（`log::debug!` / `log::warn!`）。库 crate 里**禁止** `println!`。

**注释**：说明设计原因（选择该方案的理由），而不是复述代码行为。平台相关的限制与例外应写入注释，避免后续重复排查。

## 测试

每次改动都必须通过：

```bash
cargo check --workspace --all-targets        # 零 error、零 warning
cargo test --workspace                       # 全绿
./scripts/test-linux-mock.sh                 # D-Bus mock 夹具
```

跨平台卫生：

```bash
cargo check -p uda-platform-windows \
  --target x86_64-pc-windows-gnu --all-targets
```

:::caution[Windows crate 的测试在 Linux 上不运行]
`uda-platform-windows` 是 `#![cfg(windows)]`，写在该 crate 里的 `#[test]` 在 Linux 上编译为空，等同于无人验证。

因此：**平台无关的逻辑（XML 构造、路径规范化、状态机、字节序换算）必须放在 `uda-core`**，只把真正调用 Win32/WinRT 的代码留在平台 crate。

该规则源于一次实际缺陷：Windows 的 toast XML 构造逻辑此前位于平台 crate，导致「toast 文档被操作中心静默丢弃」的问题在 CI 中无法被发现。
:::

### D-Bus mock

D-Bus 相关测试跑在 `dbus-run-session` + `python3-dbusmock` 夹具中，不需要真实桌面环境。**不要**在测试里启动真实的服务或触发真实副作用（尤其是电源动作——自动化测试绝不能锁屏、注销或关机）。

## 提交前自查

- [ ] `cargo check --workspace --all-targets` 零 warning
- [ ] `cargo test --workspace` 全绿
- [ ] `./scripts/test-linux-mock.sh` 通过
- [ ] Windows gnu 交叉编译通过（若改动了平台 crate）
- [ ] 新功能有对应单测，且该测试**真的会在 CI 上运行**
- [ ] 若改了平台行为，同步更新 `docs/internals/*_specs.md`
- [ ] `.tmp/` 下无遗留的临时脚本或日志

## 目录职责

| 路径 | 放什么 |
|------|--------|
| `crates/uda-core` | trait、类型、能力位、错误、平台无关逻辑 |
| `crates/uda-platform-linux` | D-Bus / Portal / CLI 调用 |
| `crates/uda-platform-windows` | Win32 / COM / WinRT 调用（cfg 门控） |
| `crates/uda-ffi` | C-ABI 边界、panic 遏制 |
| `crates/uda-cli` | 本地人工诊断 |
| `docs/internals/` | 协议规范字典（事实来源） |
| `examples/` | 单功能示例，三语言对齐 |

## 路线图

当前目标见 [`AGENTS.md`](https://github.com/UniDesktop/SDK/blob/develop/AGENTS.md) 的 Phased Roadmap。Phase 2（托盘、媒体、会话）已随 v0.2.0 收官，Phase 3（全局快捷键、高级剪贴板、音频端点路由、显示器亮度）为当前进行中目标。
