# RocketCompanion 修复报告

基于 `alpha` 分支最新提交 (`c61ffe1`)，已按功能解耦拆分为两个独立分支并完成 GitHub Verified Commit 推送。

---

## Branch 1: `z/agy/rc/notification-collapse-toggle`
**Commit:** [`d3fb50c8`](https://github.com/Antigravi-ty/RocketCompanion/commit/d3fb50c85b5d26697875c92e8fbe5601351294ef) (`verified: True`)

### 修改内容
1. **全局强制收纳状态 (`isForceCollapsed`) 与 Store 同步拆分**
   - `src/ui-core/stores/notificationTypes.ts`：新增 `isForceCollapsed: boolean` 状态以及 `toggleForceCollapse`、`setForceCollapsed` 动作接口。
   - `src/ui-core/stores/quickNotificationSync.ts`（新增模块）：将 `crossClientSync` 跨端同步与 `ui:expand-quick-notification` 事件监听从 `quickNotificationStore.ts` 中拆分出来，保持单文件体积在 Radar 阈值内（`15.4 KB / 438L`），避免超大文件。
   - `src/ui-core/stores/quickNotificationStore.ts`：实现 `toggleForceCollapse` / `setForceCollapsed`，并在 `expandNotificationCenter` 与 `clearAll` 时自动复位 `isForceCollapsed: false`。
2. **Notification Icon 强制收纳 / 展开交互与黄色铃铛状态**
   - `src/ui-core/navigation/QuickNotificationSystem.tsx`：
     - 当存在活跃通知（含常驻 `persistent` 通知或 `ongoing` 任务）时，点击铃铛图标会触发 `isForceCollapsed = true`，强制收纳所有展开中的通知抽屉与未读栈托盘；再次点击铃铛图标则恢复展开。
     - 当无活跃通知且存在未读栈时，保持原有点击切换未读栈托盘的行为；当通知总数归零 (`totalCount === 0`) 时自动复位 `isForceCollapsed = false`。
     - 保持 `z-[900]` 与 `pointer-events-auto` 独立层级，确保在菜单背景变暗层 (`z-40`) 之上仍可随时交互，不被菜单遮挡或锁定（区别于 `FloatingStackIcon`）。
   - `src/ui-core/navigation/QuickNotificationToastCard.tsx`：
     - 新增 `isForceCollapsed` prop，被收纳时铃铛图标 (`BellDot`) 变为黄色（浅色主题 `text-yellow-600`，深色主题 `text-yellow-400`），与系统现有警告色调保持一致。
3. **退出菜单时默认自动展开**
   - `src/ui-system/core/MenuShell.tsx`：监听 `isOpen` 由 `true -> false`（退出菜单）时自动调用 `setForceCollapsed(false)`，恢复通知默认展开状态。

### 验证结果
- `pnpm check` (`radar` + `tsc --noEmit`): **PASS (GREEN)**
- `pnpm build`: **PASS**

---

## Branch 2: `z/agy/rc/fix-transparency-glow-post`
**Commit:** [`4e9c5271`](https://github.com/Antigravi-ty/RocketCompanion/commit/4e9c52717c1027c35966b48a745e0f90ed94dac2) (`verified: True`)

### 问题根因
1. **半透明边缘遮罩写入深度缓冲，遮挡后方 Hex Glow 与透明特效**
   - 在 `DriftwoodsMaterialPaletteSpecs.ts` 中，动态遮挡网格（`Woods_Field_Wall_Posts`、`Woods_Field_Ramps_Trim`、`Woods_Field_Goal`、`Woods_Field_Goal_Posts*`、`Woods_Field_Goal_Net*`）的 Pass 2 半透明叠加层 (`OcclusionTransparentOverlay`) 处于 `renderOrder = 0.1 ~ 0.6`，但错误地开启了 `depthWrite = true`。
   - 这导致在主渲染 Pass 中，处于 `renderOrder >= 1` 的 `Woods_Field_Walls` (`1`)、`Woods_Field_Walls_Hexes` (`2`，Hex Glow 效果) 以及其他透明/发光特效在位于半透明场地边缘结构后方时深度测试失败，被错误遮挡。
2. **Bloom 后处理 (`BoostBloom`) 遮挡 Pass 未尊重动态半透明裁剪**
   - 在 `RuntimeBloomOcclusion.ts` 中，上述动态遮挡网格的 Pass 1 基础材质被整体替换为无条件写入深度的 `darkMaterial`，丢失了原着色器中 `effectiveOpacity < 0.999` 时的 `discard` 逻辑，导致半透明的场地边缘结构在 Bloom Pass 中依然充当 100% 不透明遮挡体，挡住后方的发光后处理效果；此外动态切换为 `transparent && !depthWrite` 的材质受 `renderTreeVersion` 缓存影响未被重新排除。

### 修改内容
1. `src/entities/stadium/driftwoods/DriftwoodsMaterialPaletteSpecs.ts`：
   - 将 `createDriftwoodsOcclusionOverlayPass` 中 `overlayStandardMat` 与 `overlayLambertMat` 的 `depthWrite` 设为 `false`。
   - 在 `applyDriftwoodsLayerMaterials` 中将 `isDynamicOcclusionMesh` 的深度写入修正为 `mat.depthWrite = !isOverlayMat`（Pass 1 不透明部分保持写深度，Pass 2 半透明叠加层禁止写深度）。
2. `src/entities/stadium/driftwoods/driftwoodsMeshMaterialPipeline.ts`：
   - 为 `isDynamicOcclusionMesh` 标记 `mesh.userData.isDynamicOcclusionMesh = true`，供 `RuntimeBloomOcclusion` 精确识别。
3. `src/game/runtime/RuntimeBloomOcclusion.ts`：
   - 对 `isDynamicOcclusionMesh` 不再替换为 `darkMaterial`，而是在 Bloom Pass 期间临时保存并关闭其 `colorWrite = false`，保留其原有的 `onBeforeCompile` 相机遮挡 `discard` 逻辑（不透明处继续遮挡 Bloom，半透明处放行后方 Glow/Bloom）。
   - 在 `applyBloomOccluders` 中动态检查当前材质是否已变为 `transparent && !depthWrite`，避免缓存过期导致半透明网格误写 Bloom 深度。

### 验证结果
- `pnpm check` (`radar` + `tsc --noEmit`): **PASS (GREEN)**
- `pnpm build`: **PASS**
