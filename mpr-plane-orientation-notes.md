# MPR 快速定位任意平面 – 设计与实现记录

## 1. 需求背景

- 已实现：MPR 默认视图的保存 / 恢复 / 清除（基于 `mprDefaultViewManager` + localStorage）。
- 新需求：给定一个任意平面（origin + unit normal），实现：
  - 红色 MPR 平面精确对准该平面；
  - 同时希望三视图仍保持两两正交、符合临床常见视角习惯（避免左右 / 上下 / 前后“翻转感”）。

## 2. 现有相关实现概览（basic 模式）

文件：`modes/basic/src/index.tsx`

### 2.1 常量与服务

- 常量：
  - `MPR_TOOLGROUP_ID = 'mpr'`
  - `CROSSHAIRS_TOOL_NAME = 'Crosshairs'`
  - `COMMAND_CONTEXT_ID = 'basicMprDefaults'`
- 主要服务：
  - `cornerstoneViewportService`：获取 / 操作各 viewport，相机在此设置。
  - `viewportGridService` / `displaySetService`：获取当前 viewport 的 series 信息。
  - `toolGroupService`：用于判断 viewport 是否是 MPR (`toolGroup.id === 'mpr'`)。
  - `uiNotificationService`：显示提示信息。

### 2.2 与 MPR 相关的辅助函数

- `getMprViewportIds()`：过滤出所有属于 MPR toolGroup 的 viewportId。
- `getViewportSlotId(viewportId)`：通过 `viewportInfo.getViewportOptions().id` 得到布局中的 slot id（`mpr-axial` / `mpr-sagittal` / `mpr-coronal`）。
- Crosshair 相关：
  - `getCrosshairsToolInstance()`
  - `getCrosshairWorldPos()`
  - `setCrosshairWorldPos(point: Point3)`：更新 crosshair 中心（并触发工具内部同步）。

### 2.3 自定义向量工具

- `vectorNormalize(v: Point3): Point3`
- `vectorCross(a: Point3, b: Point3): Point3`
- `vectorDot(a: Point3, b: Point3): number`
- `vectorScale(v: Point3, s: number): Point3`
- `vectorAdd(a: Point3, b: Point3): Point3`
- `vectorSub(a: Point3, b: Point3): Point3`
- `vectorLength(v: Point3): number`
- `vectorNegate(v: Point3): Point3`

## 3. 最初简单实现（仅红色视图）的局限

### 3.1 `locateViewportToPlane` 的逻辑（已被新实现替代）

- 目标：只把红色 `mpr-axial` 对准给定平面：
  1. 找到 slotId 为 `mpr-axial` 的 viewport，并确认是 `VolumeViewport`。
  2. `normalizedNormal = normalize(normal)`。
  3. `viewUp = computeViewUpVector(normalizedNormal)`：简单用 world Z/Y 与 normal 做叉乘构造任意正交向量。
  4. 若未传 `distance`，从当前相机 `position` 和 `focalPoint` 算距离，否则默认 500。
  5. `position = origin + normalizedNormal * distance`。
  6. `setCamera({ ...currentCamera, focalPoint: origin, position, viewUp })` + `render()`。
  7. 更新 Crosshair 到 `origin`。

### 3.2 结果与问题

- 红色平面几何上确实对准给定平面（origin + normal）。
- 绿色 / 黄色视图仍沿用原始 axial / sagittal / coronal 的相机方向，只是 Crosshair 移动到了新点：
  - 三个平面的 normal 不再两两正交；
  - 三视图中的十字线不再“俩俩垂直”；
  - 红色视图的 `viewUp` 为纯数学正交向量，可能与临床习惯方向差异很大，易产生“翻转感”。

结论：此实现满足“红色平面对准任意平面”的几何需求，但破坏了“三正交 + 临床习惯视角”，因此进行了重构。

## 4. Cornerstone 内部 MPR 相机规范（临床习惯的来源）

源码位置：`@cornerstonejs/core` → `packages/core/src/constants/mprCameraValues.ts`

```ts
const MPR_CAMERA_VALUES = {
  axial: {
    viewPlaneNormal: [0, 0, -1],
    viewUp: [0, -1, 0],
    viewRight: [1, 0, 0],
  },
  sagittal: {
    viewPlaneNormal: [1, 0, 0],
    viewUp: [0, 0, 1],
    viewRight: [0, 1, 0],
  },
  coronal: {
    viewPlaneNormal: [0, -1, 0],
    viewUp: [0, 0, 1],
    viewRight: [1, 0, 0],
  },
};
```

### 4.1 坐标系约定（LPS）

- X：L → R 增大
- Y：P → A 增大
- Z：F → H 增大

### 4.2 语义（医生的“直觉方向”）

- **Axial**：
  - `viewPlaneNormal = [0, 0, -1]`：从 Feet 看向 Head。
  - `viewUp = [0, -1, 0]`：屏幕“上”为 Posterior（背）。
- **Sagittal**：
  - `viewPlaneNormal = [1, 0, 0]`：从 Left 看向 Right。
  - `viewUp = [0, 0, 1]`：屏幕“上”为 Head。
- **Coronal**：
  - `viewPlaneNormal = [0, -1, 0]`：从 Front 看向 Back。
  - `viewUp = [0, 0, 1]`：屏幕“上”为 Head。

### 4.3 相关内部逻辑

- `VolumeViewport.resetCamera`：
  - 根据 `this.viewportProperties.orientation` 选择 `MPR_CAMERA_VALUES[orientation]`，重置 `viewPlaneNormal` 和 `viewUp`。
- `BaseVolumeViewport._getOrientationVectors`：
  - `setOrientation('axial' | 'sagittal' | 'coronal')` 时直接返回对应的 `MPR_CAMERA_VALUES`。
- `getCameraVectors`：
  - 从 DICOM IOP 推出 row / column / scanNormal，与 `MPR_CAMERA_VALUES` 做 best-fit，对齐方向和符号，保证与标准 MPR 一致。

> 重要结论：医生的“正常视角”来自标准的 `viewPlaneNormal` / `viewUp` 及其符号约定，而不仅仅是“平面之间正交”这一点。如果只做数学正交而不参照这套约定，就很容易出现“整体翻转”的体验。

## 5. 最终方案：以任意平面为基准重建三正交 MPR（方案 B）

### 5.1 目标

- 输入：平面 origin（Point3）、单位法向量 normal（Point3）。
- 红色 MPR：平面精确对准 origin + normal（类似新的 axial）。
- 黄色 / 绿色 MPR：与红色平面正交，三平面两两正交。
- 每个视图的“看向方向”和“屏幕上方向”尽可能保持与原 axial / sagittal / coronal 的习惯一致，避免明显翻转。

### 5.2 几何构造步骤

设：
- `n_plane`：给定的平面法向量（归一化）。
- `axialPreferredNormal` / `sagittalPreferredNormal` / `coronalPreferredNormal`：
  - 来自当前三个 MPR viewport 的 `camera.viewPlaneNormal`，
  - 若缺失则使用 `MPR_CAMERA_VALUES` 的默认值（axial `[0,0,-1]` 等）。
- `axialPreferredUp` / `sagittalPreferredUp` / `coronalPreferredUp` 类似，来自各自 `camera.viewUp`，缺失则 fallback 到 `MPR_CAMERA_VALUES` 中的 up。

1. **新 axial 法向量 `n_axial`**
   - 归一化：`n_axial = normalize(n_plane)`。
   - 若与原 axial normal 方向相反，则取反以保持大致视角一致：
     - 若 `dot(n_axial, axialPreferredNormal) < 0` 则 `n_axial = -n_axial`。

2. **新 sagittal 法向量 `n_sagittal`**
   - 将原 sagittal normal 投影到 `n_axial` 的正交平面：
     - `sagProjLen = dot(sagittalPreferredNormal, n_axial)`；
     - `sagProjOnAxial = n_axial * sagProjLen`；
     - `n_sagittal_raw = sagittalPreferredNormal - sagProjOnAxial`。
   - 若 `|n_sagittal_raw|` 太小（退化情况），退回任意与 `n_axial` 正交的向量：
     - 选 worldX/worldY 为参考，与 `n_axial` 做叉乘得到一条正交向量。
   - 单位化：`n_sagittal = normalize(n_sagittal_raw)`。
   - 若 `dot(n_sagittal, sagittalPreferredNormal) < 0`，则 `n_sagittal = -n_sagittal`，避免左右颠倒。

3. **新 coronal 法向量 `n_coronal`**
   - `n_coronal = normalize( cross(n_axial, n_sagittal) )`，自动保证与前两个正交。
   - 若 `dot(n_coronal, coronalPreferredNormal) < 0`，则 `n_coronal = -n_coronal`，避免前后颠倒。

4. **viewUp 向量的“临床友好”计算**

函数：`computeViewUpVector(normal, preferredUp)`：

- 将 `preferredUp` 投影到 normal 的正交平面：
  - `n = normalize(normal)`；
  - `pu = normalize(preferredUp)`；
  - `projLen = dot(pu, n)`；
  - `proj = n * projLen`；
  - `tangent = pu - proj`。
- 若 `tangent` 长度太小（退化：preferredUp 接近 normal）：
  - 退回到 worldZ/worldY 与 normal 做叉乘得到任意正交向量。
- 最终 `viewUp = normalize(tangent)`。

这样保证：
- `viewUp ⟂ normal`，不会破坏相机几何；
- 同时 `viewUp` 尽量贴近原有的 `viewUp`，减少上下翻转感。

5. **相机距离与位置**

- 使用当前 axial 相机距离 `d` 作为基准，保持缩放感受一致：
  - `d = |position - focalPoint|`，若异常则回退到 500。
- 对三个平面：
  - `position_axial = origin + n_axial * d`
  - `position_sagittal = origin + n_sagittal * d`
  - `position_coronal = origin + n_coronal * d`
  - 三者 `focalPoint` 都设为 `origin`。

6. **应用到三个 MPR viewport**

- 找到 slotId 为：
  - `mpr-axial`, `mpr-sagittal`, `mpr-coronal` 的 viewportId。
- 分别 `getCornerstoneViewport` 得到 `axialVp` / `sagittalVp` / `coronalVp`。
- 对每个 viewport：
  - 取当前 `camera`，构造：
    - `focalPoint = origin`
    - `position = 各自计算的 position`
    - `viewPlaneNormal = n_axial / n_sagittal / n_coronal`
    - `viewUp = axialViewUp / sagittalViewUp / coronalViewUp`
  - `setCamera(newCam)` + `render()`。
- 最后，调用 `setCrosshairWorldPos(origin)`，保证三视图交线在同一 3D 点。

### 5.3 具体实现入口

函数：`reorientMprToPlane({ origin, normal })`，位于 `modes/basic/src/index.tsx` 的 `onModeEnter` 内部作用域中。

## 6. Quick Locate 命令与测试参数

### 6.1 命令注册

在 `registerMprDefaultCommands` 中：

```ts
const quickLocateCommand = () => {
  // Hardcoded plane parameters for testing
  const origin: Point3 = [6.5853096, -152.2990733, 878.715525];
  const normal: Point3 = [-0.719653, 0.0711234, 0.6906816];

  reorientMprToPlane({
    origin,
    normal,
  });
};

commandsManager.registerCommand(COMMAND_CONTEXT_ID, 'quickLocateToPlane', {
  commandFn: quickLocateCommand,
});
```

对应的工具栏按钮 `QuickLocateToPlane` 已经配置为触发该命令。

### 6.2 预期行为

- 点击工具栏 `Quick Locate` 后：
  - 红色视图：切面通过给定 origin，法向量与给定 normal 一致（且与旧 axial 方向尽量同向）。
  - 黄色 / 绿色视图：与红色视图正交，构成新的三正交 MPR 系统。
  - Crosshair 三条线重新两两垂直；
  - 整体“病人朝向”和“屏幕上的上下 / 左右”尽量继承原 axial / sagittal / coronal 习惯，不会出现明显的整体翻转。

## 7. 未来调试与优化时的关注点

1. **若医生反馈“左右/上下反了”**：
   - 检查 `nAxial / nSagittal / nCoronal` 与对应 `*_PreferredNormal` 的点积符号逻辑是否正确；
   - 检查 `computeViewUpVector` 是否频繁走退化分支（preferredUp 接近 normal，导致使用 fallback）。

2. **若只想验证几何正确性（不考虑临床习惯）**：
   - 可以在 `reorientMprToPlane` 中暂时移除“与 preferred normal 对齐符号”的逻辑；
   - 或把 `computeViewUpVector` 改回简单 `cross` 版本，看三平面的几何关系是否正确。

3. **plane 参数来源改变时**：
   - 确保 origin / normal 所在坐标系与 Volume world 坐标一致（同为 LPS）。
   - normal 可以不是严格单位长度，但方向必须正确，函数内部会做归一化。

这份文档记录了：需求背景、Cornerstone 内部轴向与相机规范、初版实现的问题、以及“临床友好版三正交重建”方案和关键实现细节。后续若出现相关 bug 或需要继续优化，可以直接参考此文档与当前代码状态。
