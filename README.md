# macOS 原生菜单项状态能力与 Unified Tray 五态需求对比报告

## 1. macOS 原生 `NSMenuItem` / `CheckMenuItem` 支持的状态
在 macOS AppKit (`NSMenuItem`) 与 Tauri (`tauri::menu::CheckMenuItem`) 中，菜单项左侧的原生勾选栏（`state: NSControlStateValue`）**仅支持 3 种原生状态**：
1. **`NSControlStateValueOn`（`✓` 对勾）**：表示激活/选中。
2. **`NSControlStateValueMixed`（`-` 横杠）**：表示中间/部分状态。
3. **`NSControlStateValueOff`（空白）**：表示未激活/未选中。

> 原生勾选栏**不提供**空心圆（`○`）、红叉（`×`）或进程异常/后台运行等扩展状态图标。

## 2. Unified Tray 当前使用的 5 种状态
Unified Tray 需要清晰区分每个子应用的完整生命周期与窗口可见性，共使用 **5 种状态**：
1. **Stopped（未启动）**
2. **Starting / Stopping（启动中 / 停止中）**
3. **Running Foreground（运行中 · 前台可见）**
4. **Running Background（运行中 · 后台保活/托盘隐藏）**
5. **Crashed（异常崩溃）**

## 3. 结论与建议
- **为何无法直接只用原生 `CheckMenuItem`**：原生仅 3 态（`✓` / `-` / 空白），无法区分「未启动」与「崩溃」、也无法区分「前台运行」与「后台保活」，属于原生能力不完整支持。
- **当前纯文本前缀的问题**：macOS 菜单默认使用比例字体（SF Pro），`"  "`、`-`、`✓`、`○`、`×` 字符宽度不一致，导致菜单标题首字符无法垂直对齐。
- **推荐落地方案**：改用 Tauri 原生 **`IconMenuItem`**，在左侧原生图标位放置 **16×16 固定像素宽度的 RGBA 状态图标**（完全脱离文本字符宽度影响，5 种状态 100% 像素级对齐）。
