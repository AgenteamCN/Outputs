# RocketCompanion 架构重构方向与 MVP-1 实施计划

> 会话频道：`ch_8f6b8b49`
> 讨论结论：从补丁式修 Bug 转向 **Authoritative Core (Rust/WASM) + Stateless Thin Renderer (Three.js)** 的彻底解耦架构。

---

## 一、 核心问题诊断（11 项痛点的共同根因）

当前 `integration/vnext-core` 出现的问题（01 Freeplay 广播抖动、02 Boost 断线、03/04 音频前后台与广播延迟、05 遥测缺失、06 Camera 受 rAF 帧率耦合、07/08/09 GoalReplay 阵营/导演运镜/Anthem 不一致、10 Exhibition 广播无暂停且回放卡死、11 Training Pack 渲染与 Replay 4/5/6 HUD 左右翻转及车型碰撞箱错位），本质上都源于两个结构性耦合：

1. **模式分支泄漏到渲染/HUD/音频层**：渲染层与 UI 各自判断 `isBroadcast` / `isClient` / `isReplay` / `isExhibition`，导致状态机分叉。
2. **帧率与状态计算耦合**：`TimelineFrameBuffer` 在 60Hz 广播下触发 passthrough 退化为步进抖动；Camera 绑定 `rAF` 变帧率 `dt` 积分导致不同刷新率手感不一致；切视角 `4/5/6` 污染 `hudStore.localTeam`。

---

## 二、 终局 4 层解耦架构

1. **`SimCore`（Rust -> WASM，跑在 Dedicated Web Worker）**
   - **上游隔离铁律**：`crates/rocketsim` 保持 100% 原汁原味跟随上游（先锁定已验证手感的 fork 版本，后续可无冲突同步 `ZealanL/RocketSim v0.3.7+`）。任何扩展只写在自研 `crates/rc_core` 中。
   - **唯一职责**：接收 `CarControls` / `CoreCommand`，以固定 **120Hz** 驱动物理步进、统一规则状态机（Freeplay / Match / Exhibition / Training / GoalReplay / Pause）、120Hz 固定步长 Camera Kernel、以及黑匣子环形缓冲。
   - **坐标铁律**：`rc_core` 内部全程使用 RocketSim 原生 UE3 Z-up 坐标系，仅在导出 `RenderFrame` 给 Three.js 的唯一边界做一次 Y-up 坐标轴转换。
   - **防内存失效**：Worker 每次从 WASM 线性内存拷贝/切片出定长 `ArrayBuffer` 并 `Transfer` 给主线程，避免 `memory.grow` 导致视图 detached。
2. **`FrameSource`（极薄帧来源适配层）**
   - 统一接口：无论帧来自 **本地 Rust Worker**、**网络广播/联机流** 还是 **`.rcreplay` 文件**，下游完全无感。
3. **`ViewRender`（Three.js 纯哑巴渲染壳）**
   - 只消费 `RenderFrame`（带 `tick` 与 `teleportSeq`），做固定 2-tick 缓冲插值（`lerp` / `slerp`）。
   - 严禁退化为 passthrough；当 `teleportSeq` 跳变时强制硬切位姿并截断 `SpeedTrail` 丝带，杜绝进球回放或重置瞬移时的穿模拉丝。
   - Player Name Tag 必须在 `camera.updateMatrixWorld()` 之后同帧计算并绘制。
4. **`HUD` 与 `Audio`（独立旁路订阅者）**
   - **HUD**：只做 10~20Hz 低频或事件驱动刷新，切视角 `1..6` 只改 `cameraTargetId`，绝不改队伍归属。
   - **Audio**：只监听 `DomainEvent[]`（如 `GoalReplayWillStart(anthemId)`、`BallHit`）及载具参数，独立处理 `visibilitychange` / `focus` 恢复。

---

## 三、 分支与工具链策略

- **基线选择**：以干净的 `alpha`（或保留 `integration/vnext-core` 的完整环境工具链 `scripts/env/*`、`scripts/radar/*` 与 Tauri 启动脚本 `tauri:dev:tray`）建立全新干净分支（如 `feat/mvp1-rust-core-render-split`），**不把 `integration/vnext-core` 的复杂状态补丁带入新主干**。
- **工程护栏**：
  - `Radar` 零静默报错门禁：禁止空 `catch` 或 `catch { return 0 / null }`。
  - 保留 `scripts/runVNextIssuesReproSuite.mjs` 作为后续 Phase 的回归验收标尺。

---

## 四、 MVP-1 范围与验收标准（当前立即实施）

### MVP-1 目标
不接复杂菜单 UI、不接音频、不接网络服务器，用最干净的垂直切片验证主血管：

1. **纯净 `crates/rocketsim` + 新建 `crates/rc_core`（WASM）**
   - 120Hz 固定步进一车一球（Soccar 场地）。
   - 用 Rust 直接重写 `CleanPovCameraKernel`（支持 Car Cam / Ball Cam 切换，MVP-1 暂不做摇杆旋转 `lookX/lookY`）。
   - 输出定长二进制 `RenderFrame`（`ArrayBuffer`），含 `tick`、`teleportSeq`、球位姿、车位姿/Boost/Flags、Camera `(pos, quat, fov)`。
2. **极简统一输入（TS -> Worker）**
   - 直接对齐 RocketSim `CarControls`（5 个 `f32`：`throttle, steer, pitch, yaw, roll` + 3 个 `bool`：`jump, boost, handbrake` + 1 个 `ballCam`）。
   - 使用 `KeyboardEvent.code`（`KeyW/A/S/D/Q/E`、`ShiftLeft`、`Space`、`Period`、`/`、`ArrowLeft/Right`）输出 `-1 / 0 / +1`，未来手柄直接复用同一通道。
3. **Three.js 纯渲染引擎 + 强制同屏双画布验证**
   - **左屏（120Hz Direct）**：直接消费 Worker 发出的 120Hz `RenderFrame`。
   - **右屏（60Hz + Jitter Sim）**：刻意隔帧降频到 60Hz 并注入 5~25ms 随机抖动，喂给**同一个** `ArenaRenderer` 类实例。
   - 提供热键触发 `teleportSeq + 1` 重置球车，肉眼直接验收：**右屏 60Hz+抖动下零抽搐、Boost/Trail 不断线、瞬移重置不拉丝、Camera 脱离渲染帧率**。

---

## 五、 后续演进路线（MVP-2 ~ MVP-4）

- **MVP-2（Camera Shadow 对拍 + 黑匣子 Telemetry）**：在 Worker 后台可选挂载原版 `ref_camera.wasm` 做 `1e-5` 误差静默对拍；落地内存环形缓冲 + `F9/F10` 一键导出 `.rctelemetry`（含 Floating Debug Window）。
- **MVP-3（统一状态机 & 真实网络广播）**：在 `rc_core` 补齐 `GoalReplayWillStart(anthemId)`、Pause、Exhibition、阵营一致性，接通 Tauri Tunnel / WebSocket，彻底根治 07/08/09/10。
- **MVP-4（Replay Viewer & Audio & Loadout 收口）**：Replay 复用 `FrameSource`，`1..6` 切换不改 `localTeam`，接回标准化 Loadout 与 WebAudio 独立恢复链路，通过 `repro:vnext-issues` 全量验收。
