# Apple 设备（Web & Tauri）字体优先级与归属调查报告

## 一、 核心结论（TL;DR）

Apple 设备（macOS / iOS 上的 Safari、Chrome 及 **Tauri macOS 底层使用的 `WKWebView`**）上 **SF Mono 无法生效、整体被强行渲染为 JetBrains Mono**，并非单一原因，而是由 **两个叠加问题** 共同导致的：

1. **技术机制陷阱（WebKit / 浏览器根本不认字面量 `"SF Mono"`）**：
   - 在 macOS / iOS 上，**SF Mono 属于受限系统字体**（位于系统受保护目录，未作为普通用户字体暴露给 Web 内容）。无论是 Safari、Chrome 还是 **Tauri 的 `WKWebView`**，CSS 中写 `"SF Mono"` **几乎必然匹配失败**（除非用户自己手动在 Font Book 里额外安装过 SF Mono 字体包）。
   - 在 Apple 平台上真正能调起系统 SF Mono 的 CSS 关键字是 **`ui-monospace`**（以及部分上下文中的 `SFMono-Regular`）。
   - 然而目前项目中的 `--font-mono`、`MONO_FONT_STACK` 等栈定义把 `"JetBrains Mono"` 放在了 `ui-monospace` **之前**：
     ```css
     --font-mono: "SF Mono", "JetBrains Mono", ui-monospace, SFMono-Regular, Menlo, ...
     ```
     浏览器在 `"SF Mono"` 匹配失败后，紧接着就命中了通过 `@font-face` / `FontFace` / Google Fonts 加载进来的 `"JetBrains Mono"`，**根本轮不到后面的 `ui-monospace` 生效**。

2. **组件层强行指定 `--font-jetbrains`（`JetBrains Mono` 优先于一切）**：
   - 大量 `ui-core` 核心组件（HUD、MainMenu、PlayModeCard、NameTag 等）直接硬编码使用了 `var(--font-jetbrains)` 或 `MAIN_MENU_JETBRAINS_MONO_STACK`，而 `--font-jetbrains` 的定义本身就是 `"JetBrains Mono"` 排第一。

---

## 二、 代码现状逐层排查

### 1. 全局样式与 CSS 变量层（`src/styles/`）
- `src/styles/ui-system.css`（Web 游戏 UI 与 Tauri Configurator 共用）：
  - `--font-mono`: `"SF Mono", "JetBrains Mono", ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas, "Liberation Mono", "Courier New", monospace;`
  - `--font-jetbrains`: `"JetBrains Mono", "SF Mono", ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas, monospace;`
  - `#game-ui-viewport-host, #tauri-configurator-root` 默认使用 `var(--font-apple)`（SF Pro）。
  - `[data-ui-element="floating-window"]` 与 Tailwind `.font-mono` 被强制绑定到 `var(--font-mono) !important`。
  - 由于 Tailwind v4（`@import "tailwindcss";`）的 `font-mono` 工具类直接读取 `--font-mono`，上述 `:root` 定义覆盖了 Tailwind v4 原生自带的以 `ui-monospace` 打头的默认栈，导致全站（含 `EntryHubApp.tsx`、Tauri Configurator、`RuntimeViewportHost.tsx`）所有 `font-mono` 在 Apple 上全部跳过 `"SF Mono"` 直接落到 `"JetBrains Mono"`。
- `src/styles/ui-tuner.css`（`tuner.html` 使用）：与 `ui-system.css` 存在完全相同的 `--font-mono` 与 `--font-jetbrains` 定义。
- `src/styles/original_fonts.css` 与 `src/loader/MinimumProgressLoader.ts`：
  - 主动通过 `@font-face` 和 `new FontFace('JetBrains Mono', ...)` 加载 `/assets/fonts/jetbrains-mono-400.woff2`，确保了 `"JetBrains Mono"` 在运行时**一定处于可用命中状态**，从而截断了后续 `ui-monospace` 的回退。
- `tuner.html`：额外通过 Google Fonts 加载了 `JetBrains Mono` 100~800 全字重。

### 2. `src/ui-core/` 组件与状态层
`ui-core` 内部存在两类“导致 SF Mono 无法调起”的写法：
- **A 类：显式强绑 `var(--font-jetbrains)` / `"JetBrains Mono"` 第一位**
  - HUD：`InGameHUD.tsx`、`EdgePlayerHudStrip.tsx`、`StepperScoreboardHUD.tsx`、`CenterUpperInfoHUD.tsx`、`TakeoverCountdownHUD.tsx`、`TemporalReplayFooterHUD.tsx`
  - 菜单与卡片：`PlayModeCard.tsx`、`playModeCardHelpers.tsx`、`mainMenuStyleStore.ts`（`MAIN_MENU_JETBRAINS_MONO_STACK` 甚至在 `MAIN_MENU_APPLE_SANS_STACK` 里也把 `"JetBrains Mono"` 放在了 `"SF Mono", ui-monospace` 前面）
  - NameTag：`nameTagConfig.ts` 默认 `fontFamily: 'jetbrains-mono'`；`nameTagOffscreenRenderer.ts` 硬编码 `"JetBrains Mono", "SF Mono", ui-monospace`
- **B 类：意图用 SF Mono 优先，但写法有缺陷（`"SF Mono"` 在 `"JetBrains Mono"` 前，而 `ui-monospace` 在 `"JetBrains Mono"` 后）**
  - `QuickNotification.tsx`（`MONO_FONT_STACK`）
  - `dialogueStyles.tsx`（`MONO_FONT_STACK`）
  - `CircularBoostGauge.tsx`
  - `telemetrySparklinePainter.ts`、`EventImpulseScatterChart.tsx`
- **对照：少数真正能调起 SF Mono 的地方**
  - `index.html`、`minimumLoaderTemplate.ts`、`PacingOscilloscopeCanvas.tsx`、`SpatialAudioDebugGeometries.ts` 使用了 `ui-monospace, SFMono-Regular, "SF Mono", "JetBrains Mono", ...`
  - `nameTagConfig.ts` 在 `fontFamily === 'sf-mono'` 分支下写了 `"SF Mono", ui-monospace, "JetBrains Mono", ...` —— **只有把 `ui-monospace` 放在 `"JetBrains Mono"` 前面，Apple 设备才能真正渲染出 SF Mono。**

### 3. `src/ui-tuner/` 工作台层
- `TakeoverStageCanvas.tsx`、`CenterUpperStageCanvas.tsx`、`HudSpatialChoreographyStageCanvas.tsx`、`CenterUpperArchetypesSection.tsx`、`TakeoverArchetypesSection.tsx`、`PlayCardWatermarkAndColorTuner.tsx` 等均显式绑定 `var(--font-jetbrains)`；`DialogueSlotControlPanel.tsx`、`DialogueButtonManagerColumn.tsx` 使用 `'SF Mono', 'JetBrains Mono', monospace`（同样缺少前置 `ui-monospace`）。

---

## 三、 是谁弄坏的？（Git 历史溯源）

通过对比 Monorepo 初始化提交 `708aec9`（`RC@eb8be392 + UI@49e3e508 + Tuner@74a9604`）与后续 Tauri 相关提交（`65b23a8`、`ca3b841`）：

1. **原始病根在 Monorepo 合并前就已存在（UI 侧 + RC 侧历史遗留）**：
   - 在 `708aec9` 初始合入时，`src/styles/ui-system.css` 就已经写着：
     `--font-mono: "SF Mono", "JetBrains Mono", Menlo, ...`（当时甚至连 `ui-monospace` 都没有）。
   - `ui-core`（来自 `UI@49e3e508`）中的 `QuickNotification`、`dialogueStyles`、`mainMenuStyleStore`、`InGameHUD` 等也早在合并前就已经按 `"SF Mono", "JetBrains Mono"` 或 `var(--font-jetbrains)` 编写。
2. **Tauri 阶段没有修掉顺序，反而把 `--font-jetbrains` 正式化并接入了 `#tauri-configurator-root`**：
   - 在 `ca3b841` / `65b23a8`（`feat(tauri): ...`）中，为了给 HUD / Tauri Configurator 补齐变量，在 `ui-system.css` 中：
     - 把 `--font-mono` 改成了 `"SF Mono", "JetBrains Mono", ui-monospace, SFMono-Regular, ...`（**把 `ui-monospace` 放到了 `"JetBrains Mono"` 后面**，未能解决 Apple 命中问题）；
     - 新增了 `--font-jetbrains: "JetBrains Mono", "SF Mono", ui-monospace, ...`；
     - 将 `#tauri-configurator-root` 接入了 `ui-system.css`。
   - **并非 Tauri 端（Rust / `src-tauri`）搞坏的**，`src-tauri` 只负责创建 WebView 窗口并加载 Vite 页面，字体完全由前端 CSS 与 `ui-core` 决定。

---

## 四、 应该由谁来做？（职责边界与分工建议）

根据 `AGENTS.md` 与 `src/ui-core/AGENTS.md` 的架构边界定义，这项修复不能单靠某一方硬改，建议按以下职责清晰拆分：

1. **总控 / Rocket Companion 基础设施层（`src/styles/ui-system.css`、`src/styles/ui-tuner.css`）**：
   - **归属**：Rocket Companion 总控 / 样式底座（`src/styles/` 不属于 `ui-core` 内部组件）。
   - **职责**：
     - 修正 `--font-mono` 的栈顺序，将 `ui-monospace, SFMono-Regular` 移到 `"JetBrains Mono"` 之前：
       ```css
       --font-mono: ui-monospace, SFMono-Regular, "SF Mono", "JetBrains Mono", Menlo, Monaco, Consolas, "Liberation Mono", "Courier New", monospace;
       ```
       这样在 Apple 设备（Web + Tauri `WKWebView`）上会优先命中系统原生 SF Mono；在 Windows / Linux 上由于不存在 `ui-monospace` 对应的 SF Mono，会自动 fallback 到已加载的 `"JetBrains Mono"`。
     - 统一提供语义化的 `--font-mono` token，避免在非必要场景继续扩散 `--font-jetbrains` 变量。

2. **UI 团队 / `src/ui-core` 维护方**：
   - **归属**：`src/ui-core/`（`AGENTS.md` 明确划分 `src/ui-core` 为独立 UI 子系统，且 `ui-tuner` 不得反向侵入）。
   - **职责**：
     - 梳理产品设计意图：**哪些组件应当“Apple 优先 SF Mono、其余平台 fallback JetBrains Mono”**（如通用等宽 HUD、通知、弹窗、图表、NameTag、Menu Mono 模式），把其中的 `var(--font-jetbrains)`、`MAIN_MENU_JETBRAINS_MONO_STACK`、`MONO_FONT_STACK`、Canvas `ctx.font` 等统一改为 `var(--font-mono)` 或 `ui-monospace, SFMono-Regular, "SF Mono", "JetBrains Mono", ...`。
     - 确认是否有**极少数艺术字/特定字重场景**（例如依赖 JetBrains Mono Thin 100 的倒计时大数字 `TakeoverCountdownHUD` / `CenterUpperInfoHUD`）需要保留 JetBrains Mono（因为系统 `ui-monospace` / SF Mono 在 WebKit 中对 `100` Thin 字重的支持与 JetBrains Mono 100 表现不同，需要 UI 团队决策是否也切到 SF Mono）。

3. **Tauri 端（`src-tauri` & `mountTauriConfigurator.tsx`）**：
   - **不需要改 Rust 代码**。Tauri Configurator 直接挂载在 `#tauri-configurator-root` 下并引入 `src/styles/ui-system.css`，只要总控把 `ui-system.css` 中的 `--font-mono` 顺序修好，Tauri 桌面端和网页端会自动同步生效。
