# root.ts 深度研究文档

## 1. 场景与职责

### 1.1 文件定位
`root.ts` 是 Ink 框架的公共 API 入口模块，提供类似 ReactDOM 的渲染接口。它封装了 Ink 内部实现，为外部用户提供简洁的组件挂载、更新和卸载能力。

### 1.2 核心职责
- **同步渲染**：`renderSync` 立即挂载并渲染 React 组件
- **异步渲染**：默认导出的 `wrappedRender` 提供异步入口
- **根创建**：`createRoot` 提供 React 18 风格的 createRoot API
- **实例管理**：通过 `instances` 映射管理多输出流实例

### 1.3 使用场景
- **应用启动**：主入口调用 `render()` 挂载根组件
- **测试环境**：`renderSync` 用于同步测试场景
- **多窗口应用**：`createRoot` 创建多个独立渲染根
- **实例复用**：同一 stdout 的多次渲染复用 Ink 实例

---

## 2. 功能点目的

### 2.1 RenderOptions - 渲染选项

```typescript
export type RenderOptions = {
  stdout?: NodeJS.WriteStream      // 输出流，默认 process.stdout
  stdin?: NodeJS.ReadStream        // 输入流，默认 process.stdin
  stderr?: NodeJS.WriteStream      // 错误流，默认 process.stderr
  exitOnCtrlC?: boolean            // Ctrl+C 退出，默认 true
  patchConsole?: boolean           // 拦截 console 输出，默认 true
  onFrame?: (event: FrameEvent) => void  // 帧渲染回调
}
```

**设计目的**：
- 提供与 Ink 核心交互的配置接口
- 支持自定义输入输出流（测试、多窗口）
- 控制内置行为（Ctrl+C 处理、console 拦截）
- 帧级性能监控（onFrame 回调）

### 2.2 Instance 类型

```typescript
export type Instance = {
  rerender: Ink['render']      // 重新渲染
  unmount: Ink['unmount']      // 卸载应用
  waitUntilExit: Ink['waitUntilExit']  // 等待退出
  cleanup: () => void          // 清理实例
}
```

**设计目的**：
- 封装 Ink 实例操作，隐藏内部细节
- 提供清理接口，防止内存泄漏
- 与 ReactDOM.render 返回的实例类似

### 2.3 Root 类型

```typescript
export type Root = {
  render: (node: ReactNode) => void
  unmount: () => void
  waitUntilExit: () => Promise<void>
}
```

**设计目的**：
- React 18 createRoot 风格的 API
- 分离实例创建与渲染
- 支持同一根的多次顺序渲染

### 2.4 微任务边界保留

```typescript
// wrappedRender (行 107-121)
await Promise.resolve()  // 保留微任务边界
const instance = renderSync(node, options)

// createRoot (行 129-157)
await Promise.resolve()  // 保留微任务边界
const instance = new Ink({...})
```

**设计原因**（行 111-114）：
```typescript
// Preserve the microtask boundary that `await loadYoga()` used to provide.
// Without it, the first render fires synchronously before async startup work
// (e.g. useReplBridge notification state) settles, and the subsequent Static
// write overwrites scrollback instead of appending below the logo.
```

- 旧版本使用 `await loadYoga()` 提供微任务延迟
- 现在 Yoga 是同步加载，但需要保留延迟行为
- 防止首次渲染过早执行，导致状态未就绪

---

## 3. 具体技术实现

### 3.1 renderSync 函数（行 76-105）

**执行流程**：

1. **选项解析**（行 80）
   ```typescript
   const opts = getOptions(options)
   ```
   支持 `NodeJS.WriteStream` 或 `RenderOptions` 对象

2. **Ink 选项构建**（行 81-88）
   ```typescript
   const inkOptions: InkOptions = {
     stdout: process.stdout,
     stdin: process.stdin,
     stderr: process.stderr,
     exitOnCtrlC: true,
     patchConsole: true,
     ...opts,
   }
   ```
   应用默认值，用户选项覆盖

3. **实例获取/创建**（行 90-94）
   ```typescript
   const instance: Ink = getInstance(
     inkOptions.stdout,
     () => new Ink(inkOptions),
   )
   ```
   同一 stdout 复用现有实例

4. **初始渲染**（行 96）
   ```typescript
   instance.render(node)
   ```

5. **返回实例接口**（行 97-104）
   ```typescript
   return {
     rerender: instance.render,
     unmount() { instance.unmount() },
     waitUntilExit: instance.waitUntilExit,
     cleanup: () => instances.delete(inkOptions.stdout),
   }
   ```

### 3.2 wrappedRender 函数（行 107-123）

**默认导出**，提供异步接口：

```typescript
const wrappedRender = async (
  node: ReactNode,
  options?: NodeJS.WriteStream | RenderOptions,
): Promise<Instance>
```

**关键实现**（行 115）：
```typescript
await Promise.resolve()  // 微任务延迟
```

**日志记录**（行 117-119）：
```typescript
logForDebugging(
  `[render] first ink render: ${Math.round(process.uptime() * 1000)}ms since process start`,
)
```

### 3.3 createRoot 函数（行 129-157）

**React 18 风格 API**：

```typescript
export async function createRoot({
  stdout = process.stdout,
  stdin = process.stdin,
  stderr = process.stderr,
  exitOnCtrlC = true,
  patchConsole = true,
  onFrame,
}: RenderOptions = {}): Promise<Root>
```

**执行流程**：

1. **微任务延迟**（行 138）
   ```typescript
   await Promise.resolve()
   ```

2. **创建 Ink 实例**（行 139-146）
   ```typescript
   const instance = new Ink({
     stdout, stdin, stderr,
     exitOnCtrlC, patchConsole, onFrame,
   })
   ```

3. **注册实例映射**（行 149-150）
   ```typescript
   instances.set(stdout, instance)
   ```
   支持外部代码通过 stdout 查找实例

4. **返回 Root 接口**（行 152-156）
   ```typescript
   return {
     render: node => instance.render(node),
     unmount: () => instance.unmount(),
     waitUntilExit: () => instance.waitUntilExit(),
   }
   ```

### 3.4 辅助函数

#### getOptions（行 159-170）

```typescript
const getOptions = (
  stdout: NodeJS.WriteStream | RenderOptions | undefined = {},
): RenderOptions
```

**功能**：
- 如果参数是 `Stream`，包装为 `{ stdout }`
- 否则直接返回（应用默认值的合并在调用方）

#### getInstance（行 172-184）

```typescript
const getInstance = (
  stdout: NodeJS.WriteStream,
  createInstance: () => Ink,
): Ink
```

**功能**：
- 从 `instances` Map 查找现有实例
- 不存在时创建并注册
- 实现实例复用

---

## 4. 关键代码路径与文件引用

### 4.1 调用链

```
用户代码
    │
    ├──→ render(<App />) [默认导出，异步]
    │       ↓
    │   wrappedRender
    │       ↓
    │   await Promise.resolve()  // 微任务延迟
    │       ↓
    │   renderSync
    │       ↓
    │   getInstance → new Ink() → instance.render()
    │
    ├──→ renderSync(<App />, options) [同步]
    │       ↓
    │   (同上，跳过微任务)
    │
    └──→ createRoot(options) [React 18 风格]
            ↓
        await Promise.resolve()
            ↓
        new Ink()
            ↓
        返回 { render, unmount, waitUntilExit }
```

### 4.2 关键依赖

| 文件 | 用途 |
|------|------|
| `stream` | Stream 类型定义 |
| `./frame.js` | FrameEvent 类型 |
| `./ink.js` | Ink 类 |
| `./instances.js` | 实例映射管理 |
| `../utils/debug.js` | logForDebugging |

### 4.3 实例映射（instances）

```typescript
// instances.ts (推断)
const instances = new Map<NodeJS.WriteStream, Ink>()
export default instances
```

**用途**：
- 同一 stdout 的多次渲染复用实例
- 外部代码（如编辑器暂停/恢复）通过 stdout 查找实例
- `cleanup()` 从映射中移除

---

## 5. 依赖与外部交互

### 5.1 外部依赖

| 依赖 | 用途 |
|------|------|
| `stream` | Node.js Stream 模块类型 |

### 5.2 内部模块依赖图

```
root.ts
├── stream (Node.js)
├── ./frame.ts
├── ./ink.ts
│   ├── ./renderer.ts
│   ├── ./output.ts
│   └── ...
├── ./instances.ts
└── ../utils/debug.ts
```

### 5.3 与 Ink 类的交互

```typescript
// Ink 类的主要接口（ink.ts 推断）
class Ink {
  render(node: ReactNode): void
  unmount(): void
  waitUntilExit(): Promise<void>
  
  constructor(options: InkOptions)
}

type InkOptions = {
  stdout: NodeJS.WriteStream
  stdin: NodeJS.ReadStream
  stderr: NodeJS.WriteStream
  exitOnCtrlC: boolean
  patchConsole: boolean
  onFrame?: (event: FrameEvent) => void
}
```

---

## 6. 风险、边界与改进建议

### 6.1 已知边界情况

1. **Stream 类型检查**（行 162）
   ```typescript
   if (stdout instanceof Stream)
   ```
   - 使用 `instanceof Stream` 判断
   - 某些自定义流可能不通过此检查

2. **实例复用限制**
   - 仅基于 `stdout` 复用实例
   - 不同 `stdin`/`stderr` 但相同 `stdout` 会复用同一实例
   - 可能导致输入输出混乱

3. **微任务延迟不确定性**
   ```typescript
   await Promise.resolve()
   ```
   - 仅保证微任务边界，不保证具体延迟时间
   - 依赖事件循环状态

### 6.2 潜在风险

| 风险 | 严重程度 | 说明 |
|------|----------|------|
| 实例泄漏 | 中 | 未调用 cleanup() 时实例保留在 Map 中 |
| 流不匹配 | 低 | stdout/stdin/stderr 来自不同终端 |
| 重复初始化 | 低 | 多次调用 render() 创建多个 Ink 实例（不同 stdout） |
| 测试污染 | 中 | 全局 instances Map 可能在测试间泄漏 |

### 6.3 改进建议

1. **更严格的流验证**
   ```typescript
   // 建议：验证流的兼容性
   if (options.stdin && options.stdout) {
     // 验证是否来自同一终端会话
   }
   ```

2. **实例生命周期管理**
   ```typescript
   // 建议：自动清理不活跃的实例
   setInterval(() => {
     for (const [stdout, instance] of instances) {
       if (instance.isInactiveFor(TIMEOUT)) {
         instances.delete(stdout)
       }
     }
   }, CLEANUP_INTERVAL)
   ```

3. **增强的 Root API**
   ```typescript
   // 建议：添加更多 React 18 风格的方法
   export type Root = {
     render: (node: ReactNode) => void
     unmount: () => void
     waitUntilExit: () => Promise<void>
     // 新增
     isMounted: () => boolean
     getInstance: () => Ink | undefined
   }
   ```

4. **配置合并优化**
   ```typescript
   // 当前：多处默认值分散
   // 建议：集中配置管理
   const DEFAULT_OPTIONS: Required<RenderOptions> = {
     stdout: process.stdout,
     stdin: process.stdin,
     // ...
   }
   ```

5. **类型安全增强**
   ```typescript
   // 建议：更精确的选项类型
   type RenderOptions = 
     | { stdout: NodeJS.WriteStream }
     | { 
         stdout?: NodeJS.WriteStream
         stdin?: NodeJS.ReadStream
         // ...
       }
   ```

### 6.4 测试建议

- **单元测试**：
  - 选项解析的各种情况
  - 实例复用逻辑
  - cleanup 后重新渲染

- **集成测试**：
  - 完整渲染生命周期
  - 多实例并发
  - 流错误处理

- **兼容性测试**：
  - 不同 Node.js 版本的 Stream 行为
  - 自定义流实现
  - TTY vs 非 TTY 环境
