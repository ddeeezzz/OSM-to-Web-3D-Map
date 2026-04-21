# 沉浸式导航动画方案 (Immersive Navigation)

## 1. 背景与目标
用户希望在导航面板的“查找路线”后，提供一种“沉浸式漫游”模式。在该模式下，摄像机将从起点沿着规划好的路径自动移动到终点，模拟第一人称或无人机跟随的视觉体验。

## 2. 核心设计原则
- **非侵入式**：尽量不修改现有的 `initScene.js` 核心逻辑，而是通过扩展接口的方式接入。
- **安全性**：动画过程中需接管控制器，结束后必须可靠归还控制权。
- **平滑性**：使用样条曲线（Catmull-Rom）进行路径插值，保证视觉流畅。

## 3. 详细实施方案

### 3.1. 基础设施改造 (Infrastructure)
目前 `initScene.js` 的渲染循环是封闭的。为了驱动动画，我们需要一个“帧回调”机制。

**文件**: `app/src/three/initScene.js`

**变更**:
1.  在 `sceneContext` 中增加 `frameCallbacks` (Set<Function>)。
2.  暴露 `addFrameCallback(callback)` 和 `removeFrameCallback(callback)`。
3.  在 `render` 函数内部，在 `renderer.render` 之前，遍历执行所有回调。
    *   回调函数签名：`callback(deltaTime, elapsedTime)`。

```javascript
// 伪代码示例
const frameCallbacks = new Set();

const addFrameCallback = (fn) => {
  frameCallbacks.add(fn);
};

const removeFrameCallback = (fn) => {
  frameCallbacks.delete(fn);
};

const render = () => {
  const delta = clock.getDelta(); // 需要引入 THREE.Clock
  const elapsed = clock.getElapsedTime();
  
  frameCallbacks.forEach(fn => fn(delta, elapsed));
  
  controls.update();
  renderer.render(scene, camera);
};
```

### 3.2. 状态管理 (State)
在全局 Store 中管理漫游状态，以便 UI 和 3D 逻辑同步。

**文件**: `app/src/store/useSceneStore.js`

**新增状态**:
- `isImmersiveNavigating`: boolean (默认 false)
- `immersiveProgress`: number (0.0 - 1.0，用于潜在的进度条显示)

**新增 Action**:
- `setImmersiveNavigating(status)`
- `setImmersiveProgress(progress)`

### 3.3. 动画逻辑实现 (Logic)
创建一个自定义 Hook 封装漫游逻辑，避免污染组件代码。

**文件**: `app/src/hooks/useImmersiveNavigation.js` (新建)

**功能**:
1.  **路径生成**:
    *   监听 `store.highlightedRoutePath`。
    *   **决策理由**：在 `NavigationPanel.jsx` 的实现中，路径规划结果被写入 `highlightedRoutePath`（点数组）和 `activeRoute`（元数据），而 `store.route` 字段并未被实际使用。为了复用现有逻辑且避免冗余数据同步，直接使用 `highlightedRoutePath` 作为数据源。
    *   当进入沉浸模式时，将路径点转换为 `THREE.CatmullRomCurve3`。
    *   **高度处理**: 将所有点的高度提升（例如 +15米），形成“低空无人机”视角，避免穿墙。
2.  **动画驱动**:
    *   调用 `sceneContext.addFrameCallback`。
    *   维护本地 `progress` 变量 (0 -> 1)。
    *   速度控制：根据路径总长度计算总耗时（例如 速度 = 20米/秒）。
3.  **相机控制**:
    *   `camera.position.copy(curve.getPoint(progress))`
    *   `camera.lookAt(curve.getPoint(progress + epsilon))`
4.  **生命周期管理**:
    *   Start: `controls.enabled = false`
    *   End/Stop: `controls.enabled = true`, `removeFrameCallback`

### 3.4. UI 集成 (UI)

**文件**: `app/src/components/NavigationPanel.jsx`

**变更**:
- 在“清除路线”按钮旁增加“🎥 沉浸漫游”按钮。
- 仅当 `route` 存在且 `isImmersiveNavigating` 为 false 时显示。

**文件**: `app/src/components/ImmersiveControls.jsx` (新建，可选)
- 或者直接在 `App.jsx` 中渲染一个覆盖层。
- 显示“退出漫游”按钮。
- 显示进度条（可选）。

## 4. 调试与容错
- **防穿模**: 初期采用固定高度偏移（+15m）。如果未来需要更精细的避障，需要引入碰撞检测（过度工程，暂不考虑）。
- **异常中断**: 用户手动旋转视角、组件卸载、路线清除时，必须强制停止动画并恢复控制器。

## 5. 执行计划 (Execution Plan)

为了确保变更可控，将按以下三个阶段执行。

### 阶段一：基础设施与状态 (Infrastructure & State)
*   **目标**：打通 Three.js 渲染循环的回调接口，并建立全局状态管理。
*   **修改范围**：
    *   `app/src/three/initScene.js`：添加 `frameCallbacks` 集合及增删接口。
    *   `app/src/store/useSceneStore.js`：添加 `isImmersiveNavigating` 等状态字段。
*   **交付物**：
    *   具备 `addFrameCallback` 能力的 `sceneContext`。
    *   更新后的 Store。
*   **验收方法**：
    *   **代码审查 (Code Review)**：检查代码逻辑是否正确，无需繁琐的运行时测试。

### 阶段二：核心动画逻辑 (Core Logic)
*   **目标**：实现基于样条曲线的路径插值与相机控制逻辑。
*   **修改范围**：
    *   `app/src/hooks/useImmersiveNavigation.js` (新增)：封装所有漫游逻辑。
    *   `app/src/App.jsx`：引入并挂载该 Hook。
*   **交付物**：
    *   `useImmersiveNavigation` Hook 源码。
*   **验收方法**：
    1.  **真实数据测试**：利用 `NavigationPanel` 现有的“查找路线”功能生成真实路径。
    2.  **手动触发**：在 React DevTools 中手动修改 `isImmersiveNavigating` 为 `true`，或在控制台临时调用 Store 方法。
    3.  **视觉验证**：观察相机是否沿着规划路线平滑移动，且视角自然。

### 阶段三：UI 集成与交互 (UI Integration)
*   **目标**：在导航面板提供入口，并完善退出机制。
*   **修改范围**：
    *   `app/src/components/NavigationPanel.jsx`：添加“沉浸漫游”按钮。
    *   `app/src/App.jsx` 或新组件：添加“退出漫游”的覆盖层按钮。
*   **交付物**：
    *   完整的用户交互流程。
*   **验收方法**：
    1.  **手动全流程验证 (Manual Verification)**：
        *   打开导航面板 -> 输入起终点 -> 查找路线。
        *   点击“沉浸漫游” -> 确认相机开始自动飞行。
        *   飞行中点击“退出” -> 确认相机立即停止并恢复手动控制。
    2.  **边界测试**：
        *   在无路线时点击（按钮应禁用或不显示）。
        *   在漫游过程中切换路由或关闭面板（应自动重置状态）。
