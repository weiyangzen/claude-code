# executor.ts 研究文档

## 场景与职责

本文件实现 **CLI 环境的 `ComputerExecutor` 接口**，是 Computer Use 功能的核心执行引擎。它包装两个原生模块，提供统一的跨平台（实际 macOS-only）计算机控制抽象。

包装的原生模块：
1. `@ant/computer-use-input` (Rust/enigo) - 鼠标、键盘、前台应用
2. `@ant/computer-use-swift` - SCContentFilter 截图、NSWorkspace 应用、TCC

参考实现：Cowork 的 `apps/desktop/src/main/nest-only/computer-use/executor.ts`

## 功能点目的

### 1. 显示管理
- `prepareForAction()` - 隐藏非目标应用，激活目标应用
- `previewHideSet()` - 预览将被隐藏的应用集合
- `getDisplaySize()` / `listDisplays()` - 显示器信息
- `findWindowDisplays()` - 查找应用窗口所在显示器
- `resolvePrepareCapture()` - 解析并准备捕获
- `screenshot()` - 全屏截图（排除指定应用）
- `zoom()` - 区域截图

### 2. 键盘控制
- `key()` - 按键序列（支持修饰符，如 "ctrl+shift+a"）
- `holdKey()` - 按住按键指定时间
- `type()` - 输入文本（支持剪贴板粘贴）
- `readClipboard()` / `writeClipboard()` - 剪贴板操作

### 3. 鼠标控制
- `moveMouse()` - 移动鼠标
- `click()` - 点击（支持修饰符、单击/双击/三击）
- `mouseDown()` / `mouseUp()` - 鼠标按钮按下/释放
- `getCursorPosition()` - 获取光标位置
- `drag()` - 拖拽操作
- `scroll()` - 滚动

### 4. 应用管理
- `getFrontmostApp()` - 获取前台应用
- `appUnderPoint()` - 获取指定位置下的应用
- `listInstalledApps()` - 列出已安装应用
- `getAppIcon()` - 获取应用图标
- `listRunningApps()` - 列出运行中的应用
- `openApp()` - 打开应用

### 5. 独立导出
- `unhideComputerUseApps()` - 模块级导出，用于回合结束清理

## 具体技术实现

### 核心常量

```typescript
const SCREENSHOT_JPEG_QUALITY = 0.75
const MOVE_SETTLE_MS = 50  // 移动后稳定时间
```

### 目标尺寸计算

```typescript
function computeTargetDims(
  logicalW: number,
  logicalH: number,
  scaleFactor: number,
): [number, number] {
  const physW = Math.round(logicalW * scaleFactor)
  const physH = Math.round(logicalH * scaleFactor)
  return targetImageSize(physW, physH, API_RESIZE_PARAMS)
}
```
- 逻辑坐标 → 物理像素 → API 目标尺寸
- 预调整尺寸使 API 转码器早期返回，避免服务器端调整

### 剪贴板操作

```typescript
async function readClipboardViaPbpaste(): Promise<string>
async function writeClipboardViaPbcopy(text: string): Promise<void>
```
- 使用 macOS 原生 `pbpaste` / `pbcopy` 命令
- Electron 的 `clipboard` 模块不可用时的替代方案

### 动画移动

```typescript
async function animatedMove(
  input: Input,
  targetX: number,
  targetY: number,
  mouseAnimationEnabled: boolean,
): Promise<void>
```
- 缓出三次方曲线（ease-out-cubic）
- 60fps，距离比例持续时间（2000 px/sec），上限 0.5s
- 仅用于拖拽的 press→to 动作

### 修饰符处理

```typescript
async function withModifiers<T>(
  input: Input,
  mods: string[],
  fn: () => Promise<T>,
): Promise<T>
```
- 跟踪实际按下的修饰符
- 确保即使中途抛出，也只释放已按下的修饰符
- 使用 finally 保证释放

### 剪贴板粘贴输入

```typescript
async function typeViaClipboard(input: Input, text: string): Promise<void>
```
流程：
1. 保存用户剪贴板
2. 写入目标文本
3. **读回验证** - 确保写入成功
4. Cmd+V 粘贴
5. 等待 100ms（防止粘贴效果与恢复竞争）
6. finally 中恢复剪贴板

## CLI 与 Cowork 的差异

### 1. 无 `withClickThrough`
- Cowork 使用 `BrowserWindow.setIgnoreMouseEvents(true)` 实现点击穿透
- CLI 无窗口，点击穿透是 no-op
- `CLI_HOST_BUNDLE_ID` 永远不会匹配 frontmost

### 2. 终端作为代理 Host
- `getTerminalBundleId()` 检测运行的终端模拟器
- 传递给 `prepareDisplay`/`resolvePrepareCapture` 以豁免隐藏
- 从 `allowedBundleIds` 中剥离（通过 `withoutTerminal`）
- 截图中排除终端

### 3. 剪贴板通过 pbcopy/pbpaste
- 无 Electron `clipboard` 模块
- 使用 macOS 命令行工具

## 关键代码路径与文件引用

### 本文件导出
- `createCliExecutor(opts)` - 创建执行器实例
- `unhideComputerUseApps(bundleIds)` - 取消隐藏应用（模块级）

### 调用方
- `src/utils/computerUse/hostAdapter.ts:43` - 创建执行器
- `src/utils/computerUse/cleanup.ts:40` - `unhideComputerUseApps`
- `src/utils/computerUse/mcpServer.ts:27` - `listInstalledApps`

### 依赖文件
- `src/utils/computerUse/common.ts` - `CLI_CU_CAPABILITIES`, `getTerminalBundleId`
- `src/utils/computerUse/drainRunLoop.ts` - `drainRunLoop`
- `src/utils/computerUse/escHotkey.ts` - `notifyExpectedEscape`
- `src/utils/computerUse/inputLoader.ts` - `requireComputerUseInput`
- `src/utils/computerUse/swiftLoader.ts` - `requireComputerUseSwift`
- `src/utils/debug.ts` - `logForDebugging`
- `src/utils/errors.ts` - `errorMessage`
- `src/utils/execFileNoThrow.ts` - `execFileNoThrow`
- `src/utils/sleep.ts` - `sleep`

### 外部包
- `@ant/computer-use-mcp` - `ComputerExecutor` 接口、`API_RESIZE_PARAMS`、`targetImageSize`
- `@ant/computer-use-input` - 输入控制（Rust/enigo）
- `@ant/computer-use-swift` - 截图和应用管理（Swift）

## 依赖与外部交互

### 原生模块加载

```typescript
const cu = requireComputerUseSwift()  // 工厂时加载
const input = requireComputerUseInput()  // 首次输入调用时延迟加载
```

### 需要 drainRunLoop 的方法

所有 `@MainActor` Swift 方法和输入方法：
- `prepareForAction`
- `resolvePrepareCapture`
- `screenshot`
- `zoom`
- `listInstalledApps`
- `key`
- `holdKey`
- `type`（通过剪贴板时）

### 坐标系统

- 使用逻辑坐标（CSS 像素）
- 转换为物理像素（`logical * scaleFactor`）
- 再转换为 API 目标尺寸
- 参考：`@ant/computer-use-mcp/COORDINATES.md`

## 风险、边界与改进建议

### 已知风险

1. **原生模块加载失败**：
   - 工厂在加载失败时抛出，无降级模式
   - 需要确保依赖正确安装

2. **pbcopy/pbpaste 失败**：
   - 剪贴板操作依赖外部命令
   - 需要处理非零退出码

3. **修饰符卡住**：
   - 虽然 `withModifiers` 有保护，但极端情况下仍可能卡住
   - 考虑添加全局修饰符状态检查

4. **动画移动可靠性**：
   - 目标应用可能不处理中间位置
   - 某些 UI 元素可能错过拖拽事件

### 边界情况

1. **非 Darwin 平台**：
   - `createCliExecutor` 在非 macOS 平台抛出
   - 需要调用方前置检查

2. **终端未检测**：
   - `getTerminalBundleId()` 可能返回 null
   - 使用 `CLI_HOST_BUNDLE_ID` 作为回退

3. **剪贴板验证失败**：
   - `typeViaClipboard` 在读回不匹配时抛出
   - 防止粘贴错误内容

4. **孤儿按键释放**：
   - `holdKey` 使用 `orphaned` 标志处理超时
   - 防止超时后仍继续按键

### 改进建议

1. **错误恢复**：
   - 添加重试机制处理临时失败
   - 为关键操作（如截图）提供更详细的错误信息

2. **性能优化**：
   - 缓存 `getDisplaySize` 结果（显示器配置变化不频繁）
   - 考虑批量输入操作减少 HID 往返

3. **可观测性**：
   - 记录每个操作的耗时
   - 添加调试模式显示坐标转换过程

4. **测试覆盖**：
   - 添加单元测试模拟原生模块
   - 测试坐标转换逻辑
   - 验证修饰符处理

5. **跨平台准备**：
   - 虽然当前是 darwin-only，但可抽象平台接口
   - 为未来 Linux/Windows 支持预留扩展点

6. **用户体验**：
   - 在操作失败时提供更友好的错误消息
   - 考虑添加操作预览（如鼠标移动轨迹）
