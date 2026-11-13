# Track C (`5aab06257b65d6a39128dcb498e23eb62e794efa`) 合并前验收与实操指引报告

## 1. 一句话结论

**Track C 并不是“啥也没干”，它已经完成了核心后端/引擎/存储/Dev UI 闭环（21 个文件，+2290 / -12 行），代码静态检查、TypeScript 编译、Radar 架构红线与 Vite 构建全部通过。**

你之所以在界面上看到 **“Dynamic Training Pack” 是灰色的** 且 **导入回放后没看到入口**，是因为：
1. **主菜单入口故意/暂未解锁**：`src/ui-core/recipes/Layer3SelfTrainingRecipe.tsx` 第 66–73 行仍保留 `disabled: true`（且第 142 行直接 `return`），Track C 把入口放在了 **Dev Sandboxes（开发者沙盒）** 里，而不是面向普通玩家的 Self-Training Hub。
2. **回放入口走错了窗口**：从 Self-Training Hub 点 **Replay Viewer** 打开的是 `REPLAY_VIEWER_PRESET`；而 Track C 的捕获按钮 **`Put 4s Pre-Roll into Dynamic Training Pack (.rcd)`** 挂载在 **Dev Sandboxes -> `Replay to RCS Replay Importer`**（`ReplayToRcsImporterFloatingWindow.tsx`）里。

---

## 2. 自动化门禁与构建验证结果（已实测）

在 commit `5aab06257b65d6a39128dcb498e23eb62e794efa`（分支 `z/agy/rc/track-c-dynamic-training`）上实测：

| 检查项 | 命令 | 结果 | 说明 |
| :--- | :--- | :--- | :--- |
| **架构雷达 + 类型检查** | `pnpm run check` | **PASS (0)** | `radar.ts` 1364/1364 TS，0 循环依赖 / 0 越层引用 / 0 桶导出 / 0 中文或 Emoji 违规；`tsc --noEmit` 0 错误 |
| **Track C 改动文件格式** | `prettier --check` (21 files) | **PASS (0)** | Track C 提交的 21 个文件全部符合 Prettier 规范（注：仓库历史其他目录有 205 个未格式化文件，与本分支无关） |
| **生产构建** | `pnpm run build` | **PASS (0)** | Vite 打包通过，无编译/模块解析错误 |

---

## 3. 如何在本地手测验证 Track C（完整操作路径）

请按以下路径启动并验证功能闭环：

1. **打开回放导入器**：
   - 进入主菜单 -> **Developer / Dev Sandboxes** -> 打开 **`Replay to RCS Replay Importer`**。
2. **加载回放并定位到想练习的时刻**：
   - 在窗口中导入 `.replay` 文件，拖放时间轴或播放到目标介入帧（例如某次扑救或进攻前 4 秒窗口的结束帧）。
3. **一键捕获 4 秒 Pre-Roll 到 `.rcd` 训练包**：
   - 点击窗口最下方的青色按钮：
     **`Put 4s Pre-Roll into Dynamic Training Pack (.rcd)`**
   - 触发后会自动：
     - 调用 `RLRCReplaySessionController.extractCurrentDynamicSession()` 截取目标帧前 480 帧（4 秒 @ 120Hz）的完整物理状态与 Roster；
     - 存入 IndexedDB（`rc-dynamic-training-db`），并写入默认包 `"Captured From Replay"`；
     - 自动弹出 **`Dynamic Training Pack (.rcd)`** 悬浮窗（也可在 **Dev Sandboxes** 中直接找到 **`Dynamic Training Pack (.rcd)`** 卡片打开）。
4. **在 `Dynamic Training Pack (.rcd)` 窗口中验证**：
   - **Pack 管理**：新建 Pack、切换 Pack、导出 `.rcd` JSON / 导入 `.rcd` JSON、查看 `"Captured From Replay"` 下的 Shot 列表。
   - **Shot 编辑与单车接管（Per-Car Takeover）**：
     - 调整 `povSlotIndex`（主视角控制哪辆车）、`downCounterSeconds`（倒计时秒数）、`timeoutSeconds`；
     - 逐车配置 `as-previous` / `disappear` / `model`（指定 Bot 模型）/ `no-input` / `player`；
     - 可选开启 POV 车辆的出生点位置/朝向微扰（`optionalCarSpawnVariation`）。
   - **Pack 间调度**：`Move to` / `Duplicate to` 其他 Pack、`Delete`（带 Session 引用计数检查）。
   - **实机试玩与判定**：点击 **Play Shot**，进入 `DynamicTrainingMode`（Pre-roll 倒计时 -> 物理接管 -> 进球 / 超时判定 -> 战绩写回）。

---

## 4. 代码审查发现（合入前需注意的细节与小瑕疵）

整体架构拆解很干净（Schema/Codec、IndexedDB 存储、Slice 提取、Mode Runner、Judge、Dev UI、Bridge 全部独立成文件，且全部遵守 Radar 红线）。有 **3 个小问题** 建议在合并前或合入后顺手修掉：

1. **Self-Training Hub 入口仍被锁死（预期行为但需确认合并后是否解锁）**
   - 文件：`src/ui-core/recipes/Layer3SelfTrainingRecipe.tsx`（第 66–73 行 `disabled: true`、第 142 行早退）。
   - 建议：若本阶段仅作 Dev 内部验收，保持灰色是合理的；若希望合入 `alpha` 后直接对玩家开放，需决定是跳转到独立的 Pack 浏览页还是直接拉起 `DYNAMIC_TRAINING_PACK_PRESET`。
2. **`DynamicTrainingMode.ts` 第 193 行兜底读取 `ballY` 索引有误**
   - 代码：`const ballY = ballView?.posY ?? (sim?.state ? Number(sim.state[SIM_OFFSETS.BALL] ?? 0) : 0);`
   - 问题：`SIM_OFFSETS.BALL` 是球状态块起始偏移（`posX`），`posY` 应是 `SIM_OFFSETS.BALL + 1`。当前运行环境下 `sim?.getBallStateView?.()` 存在时不会走到兜底分支，但建议顺手修正。
3. **缺少自动化 Scenario / E2E 回归用例**
   - 本次提交未在 `src/game/scenario/` 新增 `DynamicTrainingMode` 的无头场景测试（如：构造一段 `DynamicTrainingStateSession` -> `startShot` -> 步进至进球/超时 -> 断言 `onOutcome`）。后续在 Track F `branch22` 或合入后建议补齐。

---

## 5. 是否可以开始合并？

**结论：可以合入 `alpha`（低风险）。**
- 对现有主流程的影响极小：仅在 `GameModeManager.ts` 注册了 `'dynamic-training-pack'` 模式、在 `runtimeBridgeAdapter.ts` 绑定了 Bridge、在 `ReplayToRcsImporterFloatingWindow.tsx` 底部增加了一个按钮，并对 `RLRCIntervene*` 扩展了可选的 `perSlotModes` / `perSlotModelIds` / `isCountdownPreroll` 参数（完全向后兼容原有 `nonPovMode` 行为）。
- 建议合并步骤：
  1. 按上述第 3 节路径在浏览器/Tauri 中实测一次「导入 `.replay` -> Put 4s Pre-Roll -> 调 Per-Car Takeover -> Play Shot 进球/超时」；
  2. 顺手修复 `DynamicTrainingMode.ts` 第 193 行的 `SIM_OFFSETS.BALL + 1`；
  3. 使用 `--no-ff` 合并到主干即可。
