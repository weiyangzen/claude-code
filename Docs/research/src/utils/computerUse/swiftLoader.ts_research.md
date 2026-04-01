# swiftLoader.ts 研究文档

## 场景与职责

本文件负责 **加载 `@ant/computer-use-swift` 原生模块**，提供截图和应用管理功能。与 `inputLoader.ts` 类似，但专门处理 Swift 模块的加载。

核心职责：
1. **模块缓存**：缓存加载的模块避免重复加载
2. **平台验证**：显式检查 macOS 平台
3. **类型导出**：导出 `ComputerUseAPI` 类型供其他模块使用

## 功能点目的

### 1. 模块加载
- 使用 `require()` 加载 `@ant/computer-use-swift`
- 缓存结果供后续调用
- 非 macOS 平台立即抛出

### 2. 平台检查
- 显式检查 `process.platform !== 'darwin'`
- 抛出清晰错误："@ant/computer-use-swift is macOS-only"
- 与 `inputLoader.ts` 的 `isSupported` 检查互补

### 3. 类型导出
- 导出 `ComputerUseAPI` 类型
- 供其他模块使用类型定义

## 具体技术实现

### 核心代码

```typescript
let cached: ComputerUseAPI | undefined

export function requireComputerUseSwift(): ComputerUseAPI {
  if (process.platform !== 'darwin') {
    throw new Error('@ant/computer-use-swift is macOS-only')
  }
  
  return (cached ??= require('@ant/computer-use-swift') as ComputerUseAPI)
}

export type { ComputerUseAPI }
```

### 模块路径解析

包使用环境变量 `COMPUTER_USE_SWIFT_NODE_PATH` 指定 .node 文件路径：
- 由 `build-with-plugins.ts` 在 darwin 目标上设置
- 未设置时回退到 `node_modules/prebuilds/` 路径
- 支持自定义构建和分发

### 需要 drainRunLoop 的方法

根据注释，四个 `@MainActor` 方法需要 `drainRunLoop`：
1. `captureExcluding` - 截图（排除应用）
2. `captureRegion` - 区域截图
3. `apps.listInstalled` - 列出已安装应用
4. `resolvePrepareCapture` - 解析准备捕获

这些方法分派到 `DispatchQueue.main`，在 libuv 下会挂起除非 CFRunLoop 被泵送。

## 关键代码路径与文件引用

### 本文件导出
- `requireComputerUseSwift()` - 获取或加载 Swift 模块
- `ComputerUseAPI` - 类型导出

### 调用方
- `src/utils/computerUse/drainRunLoop.ts:27` - 泵 tick 调用
- `src/utils/computerUse/executor.ts:273` - 工厂加载
- `src/utils/computerUse/executor.ts:320` - `prepareForAction`
- `src/utils/computerUse/executor.ts:346` - `previewHideSet`
- `src/utils/computerUse/executor.ts:354` - `getDisplaySize`
- `src/utils/computerUse/executor.ts:358` - `listDisplays`
- `src/utils/computerUse/executor.ts:362` - `findWindowDisplays`
- `src/utils/computerUse/executor.ts:380` - `resolvePrepareCapture`
- `src/utils/computerUse/executor.ts:409` - `screenshot`
- `src/utils/computerUse/executor.ts:432` - `zoom`
- `src/utils/computerUse/executor.ts:623` - `appUnderPoint`
- `src/utils/computerUse/executor.ts:630` - `listInstalledApps`
- `src/utils/computerUse/executor.ts:633` - `getAppIcon`
- `src/utils/computerUse/executor.ts:637` - `listRunningApps`
- `src/utils/computerUse/executor.ts:641` - `openApp`
- `src/utils/computerUse/executor.ts:656` - `unhideComputerUseApps`
- `src/utils/computerUse/hostAdapter.ts:48` - TCC 权限检查

### 依赖文件
- 无直接依赖（纯模块加载器）

### 外部包
- `@ant/computer-use-swift` - Swift 截图和应用管理模块

## 依赖与外部交互

### 原生模块
- `@ant/computer-use-swift` 是 Swift 编写的 Node.js 原生模块
- 使用 ScreenCaptureKit 进行截图
- 使用 NSWorkspace 进行应用管理
- 使用 TCC API 检查权限

### 环境变量
- `COMPUTER_USE_SWIFT_NODE_PATH` - 可选，指定 .node 文件路径

### 平台支持
- 严格 macOS-only（需要 ScreenCaptureKit 等框架）
- 显式平台检查提供清晰错误

### 与 drainRunLoop 的关系
- `@MainActor` 方法需要主队列排空
- 调用方必须使用 `drainRunLoop()` 包装这些方法
- 详见 `drainRunLoop.ts` 研究文档

## 风险、边界与改进建议

### 已知风险

1. **模块加载失败**：
   - 原生模块可能因架构不匹配加载失败
   - Swift 运行时依赖可能缺失

2. **平台检查重复**：
   - `executor.ts` 也有平台检查
   - 重复检查可能冗余但无害

3. **缓存生命周期**：
   - 模块缓存持续到进程结束
   - 无法在不重启的情况下重新加载

### 边界情况

1. **非 Darwin 平台**：
   - 立即抛出，阻止后续操作
   - 需要调用方前置检查

2. **重复调用**：
   - 首次调用后返回缓存实例
   - 无性能开销

3. **并发调用**：
   - 首次调用期间并发调用会重复加载
   - 实际场景中罕见

### 改进建议

1. **错误信息增强**：
   - 在错误中包含架构信息
   - 提供故障排除链接

2. **延迟加载优化**：
   - 当前在 `executor.ts` 工厂时间加载
   - 考虑延迟到实际需要时

3. **版本检查**：
   - 添加模块版本验证
   - 检测不兼容版本

4. **健康检查**：
   - 添加简单测试验证模块功能
   - 在启动时检测问题

5. **可观测性**：
   - 记录模块加载时间和结果
   - 监控加载失败率

6. **文档完善**：
   - 记录环境变量使用
   - 提供自定义构建指南
