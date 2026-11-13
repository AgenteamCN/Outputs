# Track C 修正与功能落地报告 (`3da79fa2e80f7d45f97514f828e4a05c26bf88a7`)

## 1. 已完成的改动

1. **将 4s Pre-Roll 捕获正式接入 `Replay Viewer`（面向对局原生 `.rcreplay.gz`）**
   - `src/replay/DynamicTrainingSliceExtractor.ts`：新增 `extractDynamicSessionFromReplayFrames()`，直接从 `StandaloneReplayViewerController` 的原生 `Float32Array` 状态帧中提取 4 秒（480 帧 @ 120Hz）窗口：
     - 完整保留 **State**（球 + 全车位置/四元数/速度/角速度/Boost/`hasFlipOrJump`/`airTimeSinceJump` 剩余 Flip 计时、`isJumping`/`isFlipping`、34 个 Boost Pad 冷却状态）+ **User Input**（`controls`）；
     - 不依赖 RLRC 第 4/5 轨（自动忽略）。
   - `src/replay/StandaloneReplayViewerController.ts`：新增 `extractCurrentDynamicSession()`。
   - `src/ui-core/stores/replayViewerStore.ts` & `src/ui-core/panels/user/ReplayViewerFloatingWindow.tsx`：在 Replay Viewer 底部增加 **`Put 4s Pre-Roll into Dynamic Training Pack (.rcd.gz)`** 按钮。
   - `src/ui-system/core/runtimeDynamicTrainingBridge.ts`：将 Replay Viewer 的捕获回调绑定到 `freeplay.standaloneReplayViewer.extractCurrentDynamicSession()`，并修正空状态提示文案（区分 `.rcreplay.gz` 与 `.replay`）。

2. **Replay Viewer 上传后缀筛选收紧（`.rcreplay.gz` / `.rcreplay`）**
   - `src/ui-core/panels/user/replayFilePicker.ts` & `src/ui-core/panels/user/replay-viewer/ReplayViewerLoadActionsHeader.tsx`：将 `accept` 收紧为 `.rcreplay.gz,.rcreplay`。
   - `src/replay/StandaloneReplayViewerController.ts`：在 `handleUploadFile()` 中拦截文件名含 `.rlrc` 的文件并提示；若用户手动去掉 `.rlrc` 后缀作为 `.rcreplay.gz` 强制导入，则可正常解析（忽略第 4/5 轨）并捕获进 `.rcd.gz`。

3. **`.rcd.gz` Gzip 压缩导出与导入（IndexedDB 存裸 Clip，导出时压缩）**
   - `src/ui-core/config/dynamicTrainingPackCodec.ts`：新增 `encodeRcdPackToGzipBytes()`（基于 `CompressionStream('gzip')`）与 `decodeRcdPackFromFileBytes()`（自动识别 `0x1f 0x8b` gzip 魔数解压）。
   - `src/ui-core/panels/dev/DynamicTrainingPackFloatingWindow.tsx` & `DynamicTrainingPackManagerSection.tsx`：导出文件扩展名改为 `.rcd.gz`，导入支持 `.rcd.gz,.rcd,.json,.gz`。
   - 架构说明：IndexedDB 内以独立 `StateSession`（RC Clip）直接存储以保证切球 0ms 响应；Pack 侧通过 `prevShotId` 链表维护顺序，仅在导出 `.rcd.gz` 时打包压缩。

4. **Dev Importer 命名修正 & 判分兜底修正**
   - 将 **`Replay to RCS Replay Importer`** 重命名为 **`Replay to RLRCS Replay Importer`**（涉及 `ReplayToRcsImporterFloatingWindow.tsx`、`devFloatingPresets.ts`、`devSandboxBenchesCatalog.ts`、`DynamicTrainingPackFloatingWindow.tsx`）。
   - `src/game/modes/dynamic-training/DynamicTrainingMode.ts` 第 193 行：修正 `ballY` 兜底索引为 `SIM_OFFSETS.BALL + 1`。

---

## 2. 验证结果

- `pnpm run check`（`radar.ts` + `tsc --noEmit`）：**PASS (EXIT 0)**
- `pnpm run build`（`vite build`）：**PASS (EXIT 0)**
- GitHub Verified Bot Commit：`3da79fa2e80f7d45f97514f828e4a05c26bf88a7`（`signature.isValid: true`）
