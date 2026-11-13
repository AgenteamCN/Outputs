# Track A (Telestrator) 方案深化讨论与交接确认清单

> 接续上一轮 `541087f` vs `3d28980` 对比结论，结合 `public/arenamesh/*.cmf` 实测数据与双向绘制需求，整理本轮最终共识与待拍板项。

---

## 一、 场地 CMF 碰撞网格调查结论（实测可行，强烈推荐）

已对 `public/arenamesh/` 与 `src/physics/RocketSimMeshLoader.ts` 进行核查：

1. **资产规模极小，解析成本几乎为零**：
   - 标准球场 `public/arenamesh/mesh_0.cmf` ~ `mesh_15.cmf` 共 16 个文件，总计仅 **149 KB**，包含 **8,020 个三角形、4,468 个顶点**（Throwback 球场为 `public/arenamesh/throwback_stadium/` 下 36 个 `.cmf`）。
   - 游戏启动初始化 RocketSim WASM 时，`fetchArenaCollisionMeshes()` 已经将这批 `Uint8Array[]` 缓存在内存中，**无需任何额外网络请求**。
2. **二进制格式极其简单**：
   - 8 字节头（`numTris: uint32`, `numVerts: uint32`）+ `numTris * 3 * uint32` 索引 + `numVerts * 3 * float32` 顶点（Bullet 物理单位，`* 50` 转为 UU 并做 Z-up -> Y-up 映射）。
   - 解析为纯内存的 Three.js `BufferGeometry` 组（不加入可见渲染层，专供 `Raycaster` 使用）耗时 **< 1 ms**。
3. **相比“GLB + 12 解析平面”的压倒性优势**：
   - **贴合真实弧面**：彻底解决 12 个直角平面在墙角、天花板交界处与视觉弧面穿模/悬空的问题；
   - **免疫干扰物**：天然不包含球、车、场外观众席、外墙装饰壳（`Woods_Field_Walls_Hexes` 等），无需维护 Mesh 黑名单，也不会再出现“从球的位置起笔”的异常；
   - **支持异形场馆**：Throwback 等特殊场馆直接复用对应的 `.cmf` 即可，无需手写解析平面；
   - **自动剔除背面**：将 CMF 内表面法线设为朝向场内并使用 `THREE.FrontSide` 做 Raycast，当相机在场外透过近侧外墙往里看时，`Raycaster` 会自动忽略背向射线的近侧墙面，直接命中距离相机最近的场内表面。

---

## 二、 线条采样、独立三角箭头、圆角与 Preset 实时联动

1. **线条本体（仅 Solid / Dashed 两种）+ 独立三角箭头**：
   - 线条继续沿用 **3D 空间距离采样 + RDP 抽稀 + Catmull-Rom 平滑**。
   - **箭头三角形（Arrowhead）不再绑定平滑曲线末端切线与固定 35uu 回溯**：
     - 三角形作为独立附件，**单独按松手落点（`endPoint`）生成**；
     - 方向由屏幕空间（或视平面投影）最后一段稳定轨迹决定，Billboard 平面法线使用 `normalize(camPos - endPoint)`（替代 `-camForward`），彻底消除大 FOV 屏幕边缘的透视斜切变形。
2. **圆角（Round Cap / Round Join）确认落地**：
   - 确认实现圆角端点与拐角扇形补齐（每条笔画仅多几十个三角形，开销可忽略），且圆角半径严格跟随当前线条粗细实时变化。
3. **样式引用化（支持 Preset 实时调参）**：
   - 笔画数据中记录 `presetIndex`（0/1/2）与 `tool`（`solid` / `dashed` / `arrow`），而非仅烘焙静态像素值。
   - 调整 Preset 粗细/透明度或箭头长宽时，实时重绘匹配笔画的 `BufferGeometry`（单条 < 0.2ms），实现所见即所得，调定后由后端/配置团队做云端配置对齐。

---

## 三、 网络同步模式重新评估（10Hz Piggyback 全量 vs 差值 + Hash）

| 方案 | 带宽消耗（单笔 0.5~2s） | 丢包/乱序容错 | 实现复杂度 | 结论 |
| :--- | :--- | :--- | :--- | :--- |
| **10Hz Piggyback 发增量 (Delta)** | 极低（~50-150 B/包） | 差。丢一包即断线，必须靠松手 Hash 校验失败后全量重传，状态机复杂 | 高（需维护点序号、缺失补齐） | 不推荐 |
| **10Hz Piggyback 发当前笔全量 (Full Draft) + 松手 Reliable 最终包** | **很低（平均 2~4 KB/s）**。坐标用 `Int16`（6B/点，1 uu 精度），法线用 Octahedral 2B 或由接收端 CMF 反推（0B），50 点约 300~400 B | **极强（天然幂等自愈）**。即使丢包，100ms 后下一帧直接完整覆盖补齐 | **极低**（无重传/乱序状态机） | **强烈推荐** |

- **松手确认建议**：松手时通过 **Reliable 信道**发送最终简化后的 `STROKE_COMMIT`（约 200~400 B，附带 `hash`）。因走 Reliable 信道必达，接收端直接用最终点替换草稿并用 `hash` 做幂等去重/一致性校验即可；若未来需要极致省流，也可仅发 `finalHash`，对端校验匹配回 `ACK(matched=true)`，不匹配再补发全量。
- **Hash 修复**：将 `arrowLengthRefPx`、`arrowWidthRefPx`、`presetIndex` 纳入 Hash，并将 `opacity` 统一按量化整数 `Math.round(opacity * 255)` 计算，避免浮点量化误差导致 Hash 误判。

---

## 四、 双向绘制（Client -> Host）、权限控制与“小眼睛”隐藏

1. **双向绘制架构（对齐 Match Input 流向，方便 Echo / Falcon 后续对接）**：
   - **Client 端**：绘制时本地先乐观渲染草稿（零延迟手感），同时通过统一的上行绘制端口发送给 Host（10Hz Draft + 松手 Commit）。
   - **Host 端**：作为房间唯一权威仲裁者，收到 Client 绘图包后先校验 **`allowClientDrawing` 动态开关**：
     - 关闭时直接丢弃/拦截；
     - 开启时分配权威 `strokeSeq` 与 `authorId`，再广播给所有 Client。
   - **Session 内动态开关**：Host 可在会话中随时开启/关闭 Client 绘图权限，状态同步给所有 Client；关闭时 Client 端工具栏画笔按钮自动禁用并中止当前草稿。
2. **工具栏“小眼睛”图标（Hide / Show Strokes）**：
   - 在左侧悬浮工具栏增加 **Eye / EyeOff** 按钮，仅控制本地 `TelestratorManager.rootGroup.visible`（**不删除笔画数据、不发送 Clear 网络消息**），可随时隐藏已绘线条并再次恢复显示。

---

## 五、 交接前仍需拍板的 6 个未决细节

1. **CMF 坐标映射验证**：实现时需确认 `.cmf` 顶点 `* 50` 到 Three.js `(X, Y, Z)` 的轴向与符号映射，确保与视觉场馆完全重合（误差 < 1 uu）。
2. **Client 绘图权限粒度**：`allowClientDrawing` 是**全局一刀切**（所有 Client 同步开/关），还是需要**按 `playerId` 白名单**单独授权给某位学员？
3. **“小眼睛”隐藏时的交互联动**：点击隐藏后，如果当前画笔是开启状态，是否**自动把画笔切为关闭**（防止用户盲画）？以及隐藏状态是否确定为**纯本地各自控制**（不随 Host 强同步）？
4. **Undo / Clear 的多人权限边界**：Client 的 `Ctrl+Z` / `Clear` 是**仅能撤销/清空自己画的线**，还是只有 Host 能全局 Clear？
5. **非同步视角下的远端线宽**：当 Client 自己飞去自由视角（与 Host 视角不同步）时，远端传来的笔画是**锁定 Host 当时的 `camPos/camFovDeg` 渲染**（当前实现），还是按 Client 当前相机距离重算？
6. **Preset 调参的作用范围**：实时拖动某个 Preset 粗细或箭头大小时，是**所有使用该 Preset 的历史笔画同步实时变化**，还是**仅影响当前/后续新画的笔画**？
