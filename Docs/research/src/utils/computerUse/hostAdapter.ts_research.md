# hostAdapter.ts 研究文档

## 场景与职责

本文件实现 **Computer Use Host Adapter**，是 `@ant/computer-use-mcp` 包与 CLI 环境之间的桥梁。它创建并缓存单例适配器，提供统一的接口供 MCP 包使用。

核心职责：
1. **单例管理**：进程生命周期内只创建一个适配器
2. **日志适配**：将包的日志接口适配到 CLI 的调试日志
3. **执行器工厂**：创建 CLI 特定的执行器实例
4. **权限检查**：检查 TCC 权限（Accessibility 和 Screen Recording）
5. **功能门控集成**：连接 GrowthBook 门控系统

## 功能点目的

### 1. DebugLogger 类
实现 `@ant/computer-use-mcp` 的 `Logger` 接口：
- `silly` / `debug` / `info` / `warn` / `error`
- 使用 `util.format` 支持格式化字符串
- 映射到 `logForDebugging` 的不同级别

### 2. 单例适配器 (`getComputerUseHostAdapter`)

适配器属性：
- `serverName`: `COMPUTER_USE_MCP_SERVER_NAME` ('computer-use')
- `logger`: `DebugLogger` 实例
- `executor`: 通过 `createCliExecutor` 创建
- `ensureOsPermissions`: 检查 TCC 权限
- `isDisabled`: 基于 `getChicagoEnabled()`
- `getSubGates`: 返回子功能门控
- `getAutoUnhideEnabled`: 始终返回 true（无用户偏好禁用）
- `cropRawPatch`: 始终返回 null（CLI 无同步图像处理）

### 3. TCC 权限检查

```typescript
ensureOsPermissions: async () => {
  const cu = requireComputerUseSwift()
  const accessibility = cu.tcc.checkAccessibility()
  const screenRecording = cu.tcc.checkScreenRecording()
  return accessibility && screenRecording
    ? { granted: true }
    : { granted: false, accessibility, screenRecording }
}
```

### 4. 像素验证回退

```typescript
cropRawPatch: () => null
```

**原因**：
- 包要求同步返回（直接比较 `patch1.equals(patch2)`）
- Cowork 使用 Electron 的 `nativeImage`（同步）
- CLI 的 `image-processor-napi` 基于 sharp，仅异步
- 返回 null → 验证跳过 → 点击继续（设计回退）

## 具体技术实现

### 单例模式

```typescript
let cached: ComputerUseHostAdapter | undefined

export function getComputerUseHostAdapter(): ComputerUseHostAdapter {
  if (cached) return cached
  cached = { /* ... */ }
  return cached
}
```

**设计说明**：
- 进程生命周期单例
- 原生模块在此加载（通过 `createCliExecutor`）
- 加载失败时抛出，无降级模式

### 执行器选项

```typescript
executor: createCliExecutor({
  getMouseAnimationEnabled: () => getChicagoSubGates().mouseAnimation,
  getHideBeforeActionEnabled: () => getChicagoSubGates().hideBeforeAction,
})
```

使用 getter 函数而非静态值，支持动态门控切换。

### 日志级别映射

| 包级别 | CLI 级别 |
|--------|----------|
| silly  | debug    |
| debug  | debug    |
| info   | info     |
| warn   | warn     |
| error  | error    |

## 关键代码路径与文件引用

### 本文件导出
- `getComputerUseHostAdapter()` - 获取或创建适配器单例

### 调用方
- `src/utils/computerUse/wrapper.tsx:235` - `bindSessionContext` 参数
- `src/utils/computerUse/mcpServer.ts:63` - 获取适配器

### 依赖文件
- `src/utils/computerUse/common.ts` - `COMPUTER_USE_MCP_SERVER_NAME`
- `src/utils/computerUse/executor.ts` - `createCliExecutor`
- `src/utils/computerUse/gates.ts` - `getChicagoEnabled`, `getChicagoSubGates`
- `src/utils/computerUse/swiftLoader.ts` - `requireComputerUseSwift`
- `src/utils/debug.ts` - `logForDebugging`

### 外部包
- `@ant/computer-use-mcp/types` - `ComputerUseHostAdapter`, `Logger`

## 依赖与外部交互

### 原生模块加载
- 通过 `createCliExecutor` 加载 `@ant/computer-use-input` 和 `@ant/computer-use-swift`
- 在首次调用 `getComputerUseHostAdapter()` 时加载
- 失败时抛出错误

### TCC 权限
- Accessibility：控制鼠标和键盘
- Screen Recording：捕获屏幕截图
- 通过 Swift 模块的 `tcc.checkAccessibility()` 和 `tcc.checkScreenRecording()`

### GrowthBook 集成
- `isDisabled` 回调读取 `getChicagoEnabled()`
- `getSubGates` 回调读取子功能门控
- 支持动态配置更新

## 风险、边界与改进建议

### 已知风险

1. **单例加载失败**：
   - 原生模块加载失败会导致整个 CU 功能不可用
   - 需要确保依赖正确安装

2. **像素验证缺失**：
   - `cropRawPatch` 返回 null，跳验证
   - 子门控默认关闭，风险可控

3. **权限检查时机**：
   - `ensureOsPermissions` 在需要时才调用
   - 但用户可能在会话中途撤销权限

### 边界情况

1. **重复调用**：
   - 单例模式确保只创建一个适配器
   - 多次调用返回相同实例

2. **配置变化**：
   - `isDisabled` 和 `getSubGates` 使用 getter
   - 支持 GrowthBook 动态更新

3. **非 Darwin 平台**：
   - `createCliExecutor` 内部检查平台
   - 非 macOS 抛出错误

### 改进建议

1. **懒加载优化**：
   - 考虑延迟加载 Swift 模块到实际需要时
   - 当前在工厂时间就加载

2. **错误处理**：
   - 添加更详细的加载失败诊断
   - 提供用户友好的错误消息

3. **权限引导**：
   - 在权限缺失时提供引导 UI
   - 直接打开系统偏好设置

4. **可观测性**：
   - 记录适配器创建时间
   - 监控原生模块加载成功率

5. **测试覆盖**：
   - 模拟原生模块加载失败
   - 测试配置变化传播

6. **像素验证替代方案**：
   - 探索异步像素验证的可能性
   - 或集成其他同步图像处理库
