# 阶段性复盘报告：流程观察、已完成清单与待办需求（09:56Z – 13:31Z）

- **时间范围**：2026-11-23T09:56Z – 13:31Z（共 409 条消息，其中用户消息 147 条）
- **仓库状态**：
  - `Antigravi-ty/agyDevKit`（分支 `a2` @ `4f6a0ea`，本阶段新增 **25 个 Verified Bot Commits**）
  - `Antigravi-ty/agyWorker`（分支 `main` @ `d29118e`，已线上部署）

---

## 一、流程观察与协作问题复盘

### 1. 做得好的地方（Keep）
- **Chief + `coder_1` 双代理分工高效**：Chief 专注与用户高频语音对齐（"雷霆大思考"）、拆解任务、严格 Review（拦截 `--runner-only`、自定义 `.app` 目录、`tray.lock` 等边界条件）并执行 Verified Bot Commit；`coder_1` 在 `a2-*` 分支专注实现与仓库扫描，3.5 小时内稳定交付 25 个提交。
- **网关问题当场闭环**：发现 `agyWorker` 的 `excludeSender`、`mirrorToChat` 缺 `channel` 和 `POST /cmd` 阻塞问题后，快速修改并直接部署上线，随后整个控制面研发再未卡在通讯链路。
- **语音输入 + 实时纠正**：用户边口述架构思路、边在 Mac/HarmonyOS 实测反馈，UI 微调与 Tauri 权限问题基本实现分钟级闭环。

### 2. 暴露的问题与摩擦点（Fix）
1. **云端未提前启用 Rust 工具链，导致 Mac 端连续踩 3 个编译小坑**
   - 沙盒明明自带 `scripts/env/bootstrap.sh --with-rust`，但 Chief 一开始未运行，仅靠肉眼 Review Rust 代码，导致用户 Mac 端连续遇到 `build.rs` 缺失、`bundle.icon` 缺失、`image-png` feature 缺失。
   - **已解决**：已运行 `bootstrap.sh --with-rust` 并验证全部 4 个 crate 通过 `cargo check`。
2. **未隔离 `CARGO_TARGET_DIR` 导致 OverlayFS 快照超时 & `/workspace` 重置**
   - 首次跑 `cargo check` 将 `target/` 写入 `/workspace`，触发 GCS 快照上传超时，容器回退清空了 `/workspace/.chat/*` 本地日志。
   - **已解决**：改用 `CARGO_TARGET_DIR=/tmp/agy-cache/cargo-target`，OverlayFS 稳定在 **10 MB**（预算 `<= 100 MB`）。
3. **本地 `.chat` 落盘抗风险不足**
   - 容器重置后仅靠本地 `/workspace/.chat/*` 会丢上下文，需要独立的云端持久化存储（仅 Agent 可访问）。
4. **移动端/语音输入断联与轮询耐心约定**
   - HarmonyOS 浏览器切后台/语音输入偶发闪退或断联；
   - 明确后续轮询约定：**Human <-> Chief 最多等 5 轮（第 1 轮绿、2–3 轮黄、4–5 轮红）**，**Chief <-> Coder 保持高频快节奏**。
5. **Mac 端未提交锁文件阻塞 `pull --ff-only`**
   - `pnpm-lock.yaml` 改动及 Runner `Cargo.lock` 未入库曾短暂阻塞自动拉取，后续需确认是否将 Runner `Cargo.lock` 纳入仓库。

---

## 二、已完成的需求清单（Done）

### 1. Pocket Deck & 移动端体验
- **Approve 按钮交互修复**：补齐 `@keyframes spin` 转圈动画（`0de341b`），并在收到响应前保持锁态防止重复点击（`bff564b`）。
- **0° 摄像头黑色胶囊避让**：
  - 替换为可配置 `#000` 胶囊并校准默认几何参数、`999` 浮点向上取整（`5fa9a29`, `a3b9892`, `ff4b90b`）；
  - 顶栏按钮行与摄像头/胶囊中心垂直对齐（`483949a`）；
  - 新增 `capsuleMode` 切换并隐藏 `Safe Area Calibrator` 菜单入口（`fc871f1`）。
- **提示音与局域网同步**：
  - Web Audio 提示音预设 + PC 端选择通过局域网同步至移动端（`83102ac`）；
  - 复用 `AudioContext` 修复移动端播放（`1f8213d`）并增加 500ms 冷却防双端回音连响（`b8e9354`）。
- **EMQX 连接稳定性**：Mac Host `clientId` 通过 `localStorage` 持久化（`d7f5b0d`），Ping 间隔调整为 15s（`cad4721`）。

### 2. `agyWorker` 通讯网关（`main @ d29118e`）
- 移除默认 `excludeSender: ["agent"]`；
- `mirrorToChat: true` 强制校验 `channel` 参数；
- `POST /cmd` 支持双订阅 `[respTopic, chatTopic]`，用户在聊天框回复可直接 early return。

### 3. 统一控制面（Control Plane）与仓库升级
- **仓库更名**：`Antigravi-ty/agyPreviewInstant` -> `Antigravi-ty/agyDevKit`。
- **协议与共享核心**：`packages/control-protocol` + `packages/runner-core`（NDJSON Unix Socket IPC + 进程管理 + 优雅退出，提交 `d6555e3`, `b005441`）。
- **双静默 Runner（`agy-terminal` / `agy-ax-terminal`）**：
  - 本地自签名证书打包脚本 `scripts/macos/build-runners.sh` + `--install-dir` 自定义输出目录（`3dac159`, `a61950f`）；
  - 修复 `build.rs`、`bundle.icon`（`0426d00`, `14ba041`）；
  - `ActivationPolicy::Accessory` 彻底隐藏 Dock 图标（`a61950f`）；
  - 双击 `AGY Terminal.app` 自动检测 `tray.lock`，并通过 `/bin/zsh -lc` 拉起 `Unified Tray`（`53c7c22`, `e37508b`）。
- **`Unified Tray`（`apps/unified-tray`）**：
  - `tray.lock` 互斥、IPC 客户端、6 个默认预设管理 UI（`e710fb4`）；
  - Lucide `bot-message-square` 单色模板托盘图标 + `image-png` 修复（`a61950f`, `4f6a0ea`），已全部通过云端 `cargo check`。

### 4. 只读仓库调研
- 确认 `nekohasekai/open-screen-sharing` 为 **GPLv3**（不可借鉴代码）；
- 扫描 `Antigravi-ty/P2PBrowser` (`a1`) 架构（Tauri v2 + Rust `tokio` socks5/host_relay + WebRTC DataChannel）；
- 确认 `t_host_repo_whitelist` 已存在 `a2`。

---

## 三、尚未完成 / 待推进的需求清单（Pending）

### P0：眼前立即可测 & 近期迭代
1. **Mac 端实测 `4f6a0ea` Unified Tray**：
   - 验证 `bot-message-square` 托盘图标、Dock 无图标、连接两个 Runner `.sock`、`Quit` 联动关闭三进程；
   - 决定是否将 `agy-terminal` / `agy-ax-terminal` 的 `Cargo.lock` 纳入仓库。
2. **`Unified Tray` 原生菜单升级**：
   - 左侧增加单色 Unicode 状态前缀（`●` 运行 / `○` 未跑，不用 emoji）；
   - 右侧增加原生 `Submenu`（`>` 展开 `Start / Stop (SIGINT) / Restart / View Logs`）；
   - 后续可演进 Popover 悬浮窗（**明确不需要全局快捷键**）。
3. **下线 `p2p-remote-controller` 旧托盘**：
   - 待 `Unified Tray` 稳定后，注释/移除 `p2p-remote-controller` 自带旧托盘与快捷键。

### P1：稳定性与工具链补齐
4. **云端 `.chat` 持久化存储**：为 Agent 配置仅 Agent 可访问的云端存储（Worker + KV/R2/DO），防止沙盒重置丢失上下文。
5. **沙盒防呆固化**：在 `agyDevKit` 环境脚本/检查中强制或提醒 `CARGO_TARGET_DIR=/tmp/agy-cache/cargo-target`，避免再次撑爆 OverlayFS。
6. **（可选）扫描 `Rocket Companion`**：参考其环境脚本及 WebGL -> WASM 架构迁移方案。

### P2：多仓合一与跨平台大规划
7. **多仓合一（Monorepo 收编）**：先跑通 `Preview Instant` + `Remote Controller`，再依次吸纳 `QuickGitV0` -> `P2PBrowser` -> `SingboxGUIV0` 并归档旧仓库。
8. **Windows 端验证 & 傻子模式**：通过 GitHub Actions `windows-latest` 构建 Windows 版本，解决内置 Git（MinGit / gitoxide）与一键自动配置。
9. **macOS -> Windows 远程桌面**：Mac M1 Pro `VideoToolbox` 解码 + Windows `DXGI` + `NVENC HEVC`（鼠标周边高码率 ROI `qpDeltaMap`），走现有 P2P DataChannel / 菊花链。
10. **自研网络代理与规则引擎 / 录制扩展**：`Singbox` 规则引擎重写、仅代理指定（如 Git）流量、自研 MITM/Rewrite 替代 Surge，以及未来按需新增 `AGY Capture Terminal`（窗口/Safari 音频采集）。
11. **其他待核对项**：核对 6 小时前 `Outputs` 遗留需求列表、以及管理端向 Windows 设备下发控制指令的云端控制链路。
