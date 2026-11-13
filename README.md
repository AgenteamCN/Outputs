# RocketCompanion — Tauri 桌面端隔离、Rust UPnP Tunnel 与网络协议 v2.1 升级报告

- **工作分支**: `z/agy/rc/tauri-tunnel-settings`（基于 `Alpha` 最新提交 `8d19408`）
- **已验证提交 (Verified Bot Commit)**: `7b50cb0381f0c2b828c95d54855eddcf00aa1b2c` (`signature.isValid = true`, `state = VALID`)

---

## 一、已完成改动

### 1. 修复 Web 端错误显示 Desktop Launch Configurator 的问题
- **问题定位**: `DisplayTab.tsx` 中的 `Window Mode` 已通过 `isTauriRuntime()` 隔离，但 `AdvancedTab.tsx` 中的 `Desktop Launch Configurator`（`SettingActionRow`）未做环境判断，导致在浏览器 Web 端也会渲染。
- **修复内容**:
  - 在 `src/ui-core/recipes/settings/tabs/AdvancedTab.tsx` 引入 `isTauriRuntime`，并将 `Desktop Launch Configurator` 行包裹在 `{isTauriRuntime() && (...)}` 中，确保仅在 Tauri 桌面端出现。

### 2. Rust 底层 UPnP IGD + 原生 UDP Tunnel（`src-tauri`）
- **新增依赖**: `src-tauri/Cargo.toml` 引入 `igd-next = "0.18.0"`。
- **新增模块**: `src-tauri/src/tunnel_manager.rs`
  - `tauri_upnp_probe`: 异步探测局域网 UPnP IGD 网关、本地 LAN IPv4 与公网 External IP。
  - `tauri_tunnel_open`: 绑定本地原生 `UdpSocket`，可选自动执行 UPnP UDP 端口映射（3600s 租约），并启动独立接收线程将入站 UDP 数据包通过 `rc-tunnel-packet` 事件推送至前端。
  - `tauri_tunnel_connect`: 绑定远端 `ip:port` 并发送 `RCPH` 探测包协助 NAT 打洞。
  - `tauri_tunnel_send` / `tauri_tunnel_get_status` / `tauri_tunnel_close`: 提供原生 UDP 发送、状态/流量统计查询及关闭时自动释放 UPnP 端口映射。
  - `cleanup_all_tunnels`: 在 `src-tauri/src/lib.rs` 的 `tauri::RunEvent::Exit` 中统一清理所有活跃 Tunnel 线程与 UPnP 映射。
- **顺带修复跨平台编译问题**: `src-tauri/src/lib.rs` 中 `tauri::RunEvent::Reopen` 原先未加 `#[cfg(target_os = "macos")]` 导致 Linux/Windows 下 `cargo check` 报错，已补齐条件编译门控。

### 3. 网络层协议升级（`v2.1`）与 Tunnel 抽象层
- **协议版本与扩展包类型**: 新增 `src/network/protocol/TunnelProtocol.ts`
  - 定义 `NETWORK_PROTOCOL_VERSION = { major: 2, minor: 1 }`（`NETWORK_PROTOCOL_VERSION_TAG = "2.1"`）。
  - 定义 8 字节二进制信封头（`TUNNEL_ENVELOPE_MAGIC = 0x5243` `"RC"`）及扩展包类型 `TUNNEL_PACKET_TYPES`：`PHYSICS_SNAPSHOT`、`USER_INPUT`、`RELIABLE_CONTROL`、`PING_PONG_16B`、`TELESTRATOR_SYNC`、`TUNNEL_HANDSHAKE`、`TUNNEL_KEEPALIVE`、`UPNP_ENDPOINT_ADVERTISE`、`CSR_STATE_BROADCAST`、`ROOM_ACTIVITY_SWITCH`、`DIAGNOSTIC_TELEMETRY`、`CUSTOM_EXTENSION`。
- **Tunnel 抽象与自动路由**: 新增 `src/network/transport/tunnel/`
  - `TunnelTransport.ts`: 统一 `TunnelTransport` 接口（`open`、`connectToPeer`、`send`、`sendTyped`、`onPacket`、`onStateChange`、`getStats`、`close`）。
  - `RustTunnelTransport.ts`: Tauri 环境下的 Rust UPnP + UDP Tunnel 实现。
  - `WebTunnelTransport.ts`: 浏览器环境下的 WebRTC (`P2PWebRTCChannel`) 适配实现。
  - `TunnelFactory.ts`: 自动检测 `isTauriRuntime()`，有 Rust 用 `RustTunnelTransport`，无则回退 `WebTunnelTransport`。
- **会话管理器接入**: 在 `src/network/transport/MultiplayerSessionManager.ts` 暴露 `getProtocolVersion()`、`getPreferredTunnelKind()`、`probeLocalUpnp()`、`createSessionTunnel()` 供上层与 UI 联调使用。

---

## 二、Deep Scan：桌面端（Tauri/Rust）可增强体验的候选功能清单

针对你提到的“还有一个基于 Rust、让桌面端体验明显大于网页端的功能，一时想不起来了”，我们对 `src-tauri/` 与 `src/` 做了全量扫描，发现以下高价值候选（按与现有代码关联度排序）：

1. **原生手柄高帧率轮询与后台输入（`gilrs` / 原生 HID）** — **最可能**
   - 现状：`src/input/GamepadController.ts` 依赖浏览器 `navigator.getGamepads()`，受限于浏览器 `rAF` 帧率、窗口失焦冻结以及部分手柄（DualSense/Xbox 扩展按键）兼容性。
   - Rust 方案：在 Rust 侧通过 `gilrs` 以 250Hz~1000Hz 独立线程轮询手柄并注入输入状态，彻底摆脱浏览器焦点限制与帧率抖动。
2. **本地 Workspace 文件系统直连（`replays/` 与 `training_packs/`）** — **已半成品**
   - 现状：`src-tauri/src/config_store.rs` 已经创建了 `workspace/replays/` 和 `workspace/training_packs/` 目录，但尚未与前端 `src/replay/` 的导入导出打通。
   - Rust 方案：提供原生文件监听/读写命令，甚至自动监听本机 Rocket League `Demos/` 目录自动导入 `.replay` 文件，无需每次手动拖拽或文件弹窗。
3. **Discord Rich Presence（`discord-rich-presence`）**
   - 在桌面端展示当前模式（Freeplay / Online Match / Replay Viewer）、房间人数与时长。
4. **全局快捷键与无边框/置顶热键（`tauri-plugin-global-shortcut`）**
   - 在托盘静默模式或后台训练时支持全局呼出、快速录屏/标记书签。
5. **已实现项确认**: `tauri-plugin-macos-fps`（macOS ProMotion 120Hz+ rAF 解锁）及系统托盘/启动配置器（`tray_manager.rs`、`window_manager.rs`）已在仓库中完成。

---

## 三、验证结果

- `pnpm run check`（`radar` 架构边界检查 + `tsc --noEmit`）：**PASS (GREEN)**，1392/1392 文件 100% TS，0 循环依赖，0 边界违规。
- `pnpm run build`（Vite 生产构建）：**PASS**（28.46s 完成）。
- `cargo check`（`src-tauri`）：**PASS**（0 errors，已完成系统工具链 `gcc`、`pkg-config`、`libwebkit2gtk-4.1-dev` 等依赖安装并编译通过）。

---

## 四、后续分支与 PR 建议

- 当前所有底层改动已集中在单分支 `z/agy/rc/tauri-tunnel-settings`，且按职责拆分为独立低耦合模块（UI 门控、Rust `tunnel_manager`、协议 `TunnelProtocol`、传输抽象 `tunnel/`），可直接用于联调验证与发起 PR。
- 待你确认上述 Deep Scan 中的第 3 项桌面端增强功能（例如原生手柄轮询或本地 Replay 目录直连）后，可继续在此分支或新开独立 `z/agy/rc/*` 分支推进。
