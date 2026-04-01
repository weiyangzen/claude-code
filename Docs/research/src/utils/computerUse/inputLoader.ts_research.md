# inputLoader.ts 研究文档

## 场景与职责

本文件负责 **延迟加载 `@ant/computer-use-input` 原生模块**，提供输入控制（鼠标、键盘、前台应用）功能。采用延迟加载模式优化性能，确保仅在需要时才加载 Rust/enigo 原生模块。

核心职责：
1. **模块缓存**：缓存加载的模块避免重复加载
2. **平台检查**：验证平台支持（darwin-only）
3. **类型收窄**：将联合类型收窄为具体的 API 类型

## 功能点目的

### 1. 延迟加载
- 首次调用 `requireComputerUseInput()` 时才加载模块
- 缓存结果供后续调用使用
- 截图专用流程（不调用输入方法）永远不会加载 enigo .node

### 2. 平台验证
- 检查 `input.isSupported` 标志
- 不支持时抛出清晰错误
- 与 `swiftLoader.ts` 的平台检查互补

### 3. 类型安全
- 输入模块导出联合类型（支持/不支持）
- 本文件负责类型收窄
- 调用方获得裸 `ComputerUseInputAPI` 无需重复检查

## 具体技术实现

### 核心代码

```typescript
let cached: ComputerUseInputAPI | undefined

export function requireComputerUseInput(): ComputerUseInputAPI {
  if (cached) return cached
  
  const input = require('@ant/computer-use-input') as ComputerUseInput
  
  if (!input.isSupported) {
    throw new Error('@ant/computer-use-input is not supported on this platform')
  }
  
  return (cached = input)
}
```

### 模块路径解析

包使用环境变量 `COMPUTER_USE_INPUT_NODE_PATH` 指定 .node 文件路径：
- 由 `build-with-plugins.ts` 在 darwin 目标上设置
- 未设置时回退到 `node_modules/prebuilds/` 路径
- 支持自定义构建和分发

### 内部实现细节

根据注释，`key()` / `keys()` 方法：
1. 通过 `dispatch2::run_on_main` 将 enigo 工作分派到 `DispatchQueue.main`
2. 在 tokio worker 上阻塞通道等待完成
3. **问题**：在 libuv（Node/bun）下主队列永不排空，Promise 挂起
4. **解决**：调用方使用 `drainRunLoop()` 包装

## 关键代码路径与文件引用

### 本文件导出
- `requireComputerUseInput()` - 获取或加载输入模块

### 调用方
- `src/utils/computerUse/executor.ts:456` - `key()` 方法
- `src/utils/computerUse/executor.ts:476` - `holdKey()` 方法
- `src/utils/computerUse/executor.ts:510` - `type()` 方法
- `src/utils/computerUse/executor.ts:528` - `moveMouse()` 方法
- `src/utils/computerUse/executor.ts:545` - `click()` 方法
- `src/utils/computerUse/executor.ts:558` - `mouseDown()` 方法
- `src/utils/computerUse/executor.ts:562` - `mouseUp()` 方法
- `src/utils/computerUse/executor.ts:566` - `getCursorPosition()` 方法
- `src/utils/computerUse/executor.ts:583` - `drag()` 方法
- `src/utils/computerUse/executor.ts:600` - `scroll()` 方法
- `src/utils/computerUse/executor.ts:614` - `getFrontmostApp()` 方法

### 依赖文件
- 无直接依赖（纯模块加载器）

### 外部包
- `@ant/computer-use-input` - Rust/enigo 输入控制模块

## 依赖与外部交互

### 原生模块
- `@ant/computer-use-input` 是 Rust 编写的 Node.js 原生模块
- 使用 enigo 库进行跨平台输入模拟
- 提供鼠标、键盘、前台应用查询功能

### 环境变量
- `COMPUTER_USE_INPUT_NODE_PATH` - 可选，指定 .node 文件路径

### 平台支持
- 当前仅支持 macOS（darwin）
- `isSupported` 标志在其他平台返回 false

### 与 drainRunLoop 的关系
- 输入方法需要主队列排空
- 调用方必须在 `drainRunLoop()` 内调用输入方法
- 详见 `drainRunLoop.ts` 研究文档

## 风险、边界与改进建议

### 已知风险

1. **模块加载失败**：
   - 原生模块可能因架构不匹配（如 ARM64/x64）加载失败
   - 需要确保构建系统正确配置

2. **平台检测延迟**：
   - 延迟到首次调用才发现不支持
   - 可能在用户操作中途失败

3. **缓存生命周期**：
   - 模块缓存持续到进程结束
   - 无法在不重启的情况下重新加载

### 边界情况

1. **重复调用**：
   - 首次调用后返回缓存实例
   - 无性能开销

2. **并发调用**：
   - 首次调用期间并发调用会重复加载
   - 实际场景中罕见（通常串行初始化）

3. **截图专用流程**：
   - 只调用截图方法时永远不会加载输入模块
   - 优化内存和启动时间

### 改进建议

1. **预加载选项**：
   - 添加选项在启动时预加载模块
   - 避免首次操作延迟

2. **更详细的错误**：
   - 区分"不支持"和"加载失败"
   - 提供架构/平台信息

3. **健康检查**：
   - 添加简单测试验证模块功能
   - 在启动时检测问题

4. **热重载支持**：
   - 探索开发环境下的模块热重载
   - 便于迭代开发

5. **可观测性**：
   - 记录模块加载时间和结果
   - 监控加载失败率

6. **文档完善**：
   - 在 README 中记录环境变量
   - 提供故障排除指南
