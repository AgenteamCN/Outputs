# Automation 日志分析与修复报告

Commit: `78ae3820097912c0520f396fb14eff421fa7db70` (branch `a2`, verified bot commit)

---

## 一、问题根因定位（结合本次 Automation 导出日志）

### 1. 「针对第一个 window 重复设置」的根因
在排查底层 AX Helper 与前端状态流时，发现两个直接导致「第一个窗口反复被配置」的缺陷：

1. **Swift AX Helper 标题回退误命中第一个窗口（核心根因）**
   - 文件：`packages/macos-ax-helper/Sources/MacosAxHelper/AccessibilityEngine.swift` (`findWindowElement`)
   - 原逻辑在 `_AXUIElementGetWindow(win, &cgId) == targetWindowId` 未命中时（例如窗口位于其他 Space、最小化或刚新建尚未绑定 AX 窗口），会回退到按窗口标题精确匹配：`axWindows.first { title == info.title }`。
   - 由于所有 AI Studio 窗口的标题都是 `"Google AI Studio"`，**`axWindows.first` 永远会命中第一个窗口**，导致对其他 `windowId`（如日志中 `run-2/3/5/6` 反复出现的 `56122`）执行 `scanWindow`、`focusWindow`、`performAction` 时，实际全部打到了**第一个窗口**上。
   - **修复**：仅当 `info.title` 非空，且该标题在 `safariWindows` 与 `axWindows` 中**均唯一**时才允许标题回退，否则返回 `nil` 明确报错，杜绝静默误操作第一个窗口。
   - **注意**：由于当前沙箱为 Linux x86_64 环境，无法本地编译 Swift macOS 二进制，请在 macOS 上重新执行 `pnpm ax-helper:build` 使 Swift 修复生效。

2. **模型名称正则与界面展示名不匹配，导致 `select_model` 每次都不跳过**
   - 文件：`apps/p2p-remote-controller/src/services/aiStudioAxHelpers.ts` (`isHarnessModelSelected`)
   - 默认 `modelRegex` 为 `"gemini-4-hydrogen"`（带连字符），但收起后的模型按钮通常只包含展示名 `"Gemini 4 Hydrogen"`（空格分隔）。因此 `isHarnessModelSelected` 始终返回 `false`，导致每次初始化都会重复点开模型选择器。
   - **修复**：`buildModelMatcher` 同时支持连字符与空格分隔格式（例如 `gemini-4-hydrogen` 同时匹配 `Gemini 4 Hydrogen`）。

3. **`resetWindowNetworkConditionRef` 未绑定**
   - 文件：`apps/p2p-remote-controller/src/hooks/useSafariWindowManager.ts`
   - `resetWindowNetworkConditionRef` 初始化为空函数但从未绑定 `batchOps.resetWindowNetworkCondition`，导致页面重载或关闭时无法正确清理窗口就绪缓存。已补充 `useEffect` 绑定。

---

### 2. 「Load All Keys (Load or Keys) 点不动 / 跳过」的根因
对应日志中的 `run-7`、`run-8`、`run-9`、`run-10`、`run-11`：

1. **有 96 个元素但因 `webAreaLoaded=false` 被直接跳过**
   - 日志证据：`run-7` 与 `run-8` 中，新建的 4 个窗口（`59388, 59386, 59384, 59382`）和现有窗口均记录 `status=loading (Loading (96 elements))`。页面其实已经渲染出 96 个 AX 元素，但 Safari 的 `AXLoaded` 属性尚未置为 `true`，`linkAllSafariApiKeys` 直接因 `info.status === "loading"` 跳过。
2. **切窗口后瞬时只扫出 4 个元素未重试**
   - 日志证据：`run-10` 所有窗口记录 `Loading (4 elements)`，`linkAllSafariApiKeys` 未做短暂停顿重试就直接跳过。
3. **点击 `AXPress` 使用 `BestEffort` 吞掉异常且不校验结果**
   - 原代码使用 `performAxActionBestEffort`，即使 `AXPress` 失败也会记为 `outcome: "ok"`；且点击后不检查窗口状态变化。
   - **修复**（`apps/p2p-remote-controller/src/services/aiStudioBatch.ts`）：
     - 若首次扫描元素数 `<= 10`，等待 350ms 后重试一次扫描；
     - 若元素数 `> 10`（即使 `webAreaLoaded=false`），不再盲目跳过，继续检查 `isApiKeySelected` 与 `Link API Key` / `No API key selected` 按钮；
     - 改用 `performAxAction` 执行 `AXPress`（失败会如实记录 `failed`），并在点击 250ms 后复扫记录 `pre=(...)` 与 `post=(...)` 状态；
     - 修复 `resolveTargetWindowIds` 在 `explicitIds` 为空数组 `[]` 时错误回退到所有 Safari 窗口的问题。

---

## 二、Automation Log 细节增强

为了后续遇到任何 UI/AX 异常时能够直接从导出日志定位，增强了以下细节：
1. **`summarizeScanForTracker`** (`apps/p2p-remote-controller/src/services/automationTracker.ts`)：
   - 新增 `url`（pathname）、`skipBtn=true`、`noThanks=true`、`harnessBtn="..."`（模型按钮文本摘要）、`geminiDefault=true`、`saveBtn=enabled|disabled|missing`。
2. **`init_all` 队列明细** (`apps/p2p-remote-controller/src/services/aiStudioBatchRoundRobin.ts`)：
   - 为每个进入 `pendingQueue` 的窗口记录 `queue_window` (`started`)，包含 `status`、`apiKey`、`model`、`net` 布尔值与 `detail`，明确说明为何该窗口被重新排队。
3. **`select_model` 启动日志** (`apps/p2p-remote-controller/src/services/aiStudioAxHelpers.ts`)：
   - 未跳过模型选择时记录 `select_model` (`started`) 及当前 `harnessBtn` 文本。
4. **`save_network_modal` 失败复扫** (`apps/p2p-remote-controller/src/services/aiStudioInitSteps.ts`)：
   - 针对日志 `run-12` 中 `59424` 出现的 `Network Configuration modal did not close after clicking Save`，在失败时自动复扫窗口并记录 `summarizeScanForTracker`（含 `saveBtn=enabled|disabled`、`domains=[...]`）。

---

## 三、验证结果

以下检查均在沙箱中运行并通过：
- `pnpm run p2p:check`（`tsc --noEmit && eslint src`）：通过，0 错误
- `pnpm --filter p2p-remote-controller exec vitest run`：10 个测试文件、37 个测试用例全部通过（含新增的 `Gemini 4 Hydrogen` 匹配测试、`summarizeScanForTracker` 测试、`linkAllSafariApiKeys` 在 `webAreaLoaded=false` 且 `elements > 10` 时的点击测试）
- `pnpm run check`（根目录 `tsc --noEmit && eslint src && vitest run`）：3 个测试文件、19 个测试用例全部通过，0 错误

---

## 四、环境初始化脚本在当前沙箱中的局限性（重要说明）

1. **`/tmp` 在每次独立命令调用间会重置**
   - 本次通过 `bash scripts/env/bootstrap.sh` 完成初始化后，在下一次独立 shell 调用中 `/tmp/agy-cache/` 已被清空，导致 `apps/p2p-remote-controller/node_modules -> /tmp/agy-cache/p2p-node_modules` 悬空，`tsc --noEmit` 报找不到 `react` 等模块。
   - 初始化脚本无法跨独立容器调用阻止 `/tmp` 重置；实际执行任何 `pnpm` 检查或测试命令前，必须在**同一条 shell 命令中**串联 `node scripts/env/doctor.mjs --auto-restore && <command>`。
2. **Rust / Tauri 工具链与 macOS Swift 编译**
   - `bootstrap.sh` 默认不安装 Rust（`[WARN] Rust/Tauri (optional) not installed`）；且当前沙箱为 Ubuntu 24.04 x86_64，无法编译 `packages/macos-ax-helper` 的 Swift macOS 代码（`AccessibilityEngine.swift` 需在 macOS 本机执行 `pnpm ax-helper:build` 重新打包）。
