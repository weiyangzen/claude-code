# workloadContext.ts 研究文档

## 场景与职责

`workloadContext.ts` 提供基于 `AsyncLocalStorage` 的 turn-scoped 工作负载标签机制。用于区分不同来源的请求（如 cron 任务 vs 普通用户交互），以便进行差异化的权限控制和审计。

**设计背景（来自代码注释）：**
- 与 `bootstrap/state.ts` 分离，因为 bootstrap 被浏览器 SDK 入口点 transitively 导入，而浏览器环境无法使用 Node.js 的 `async_hooks`
- 使用 `AsyncLocalStorage` 而非全局可变槽位，因为 void-detached 后台 agent 在第一个 await 处 yield，父 turn 的同步 continuation（包括 `finally` 块）会在 detached closure 恢复前完成执行

**核心使用场景：**
- Cron 任务执行时标记工作负载为 `'cron'`
- 区分自动化任务和人工交互
- 服务器端请求来源追踪

## 功能点目的

### 1. 工作负载标签获取 (`getWorkload`)
- **目的**：获取当前异步上下文中的工作负载标签
- **返回**：`'cron'` 或 `undefined`

### 2. 工作负载上下文包装 (`runWithWorkload`)
- **目的**：在指定的工作负载上下文中执行函数
- **关键设计**：始终建立新的上下文边界，即使 `workload` 为 `undefined`
- **原因**：防止已泄漏的 cron 上下文通过 pass-through 传播

## 具体技术实现

### 核心数据结构

```typescript
// 服务器端允许的 workload 值
// 由 _sanitize_entrypoint 在 claude_code.py 中验证
// 只接受小写 [a-z0-9_-]{0,32}，大写字母会在第 0 字符处停止解析
export type Workload = 'cron'
export const WORKLOAD_CRON: Workload = 'cron'

// AsyncLocalStorage 实例
const workloadStorage = new AsyncLocalStorage<{
  workload: string | undefined
}>()
```

### 关键流程

```
getWorkload() → string | undefined
└── workloadStorage.getStore()?.workload

runWithWorkload<T>(workload, fn) → T
└── workloadStorage.run({ workload }, fn)
    └── 在新的 ALS 上下文中执行 fn
        └── fn() 内部的 getWorkload() 返回传入的 workload
```

### 上下文隔离机制

```typescript
// 问题场景：泄漏的 cron 上下文
// REPL: queryGuard.end() → _notify() → React subscriber 
// → scheduled re-render captures ALS at scheduling time 
// → useQueueProcessor effect → executeQueuedInput → here

// 错误实现（pass-through）
if (!workload) {
  return fn()  // 允许父上下文泄漏！
}

// 正确实现（始终新建边界）
return workloadStorage.run({ workload }, fn)
// 即使 workload 是 undefined，也能确保 getWorkload() 返回 undefined
```

## 关键代码路径与文件引用

### 导出类型和常量
- `src/utils/workloadContext.ts:25` - `Workload` 类型
- `src/utils/workloadContext.ts:26` - `WORKLOAD_CRON` 常量

### 导出函数
- `src/utils/workloadContext.ts:32` - `getWorkload()`
- `src/utils/workloadContext.ts:52` - `runWithWorkload<T>()`

### 依赖
| 依赖 | 用途 |
|------|------|
| `async_hooks` (Node.js) | `AsyncLocalStorage` 类 |

### 调用方（预期）
- Cron 任务执行器
- Agent 工具执行器
- 需要区分请求来源的其他模块

## 依赖与外部交互

### 外部依赖
```typescript
import { AsyncLocalStorage } from 'async_hooks'
```

### 内部依赖
无内部依赖。

### 服务器端验证
- Python 后端 `_sanitize_entrypoint` 函数验证 workload 值
- 只允许小写字母、数字、下划线、连字符
- 最大长度 32 字符

## 风险、边界与改进建议

### 已知风险

1. **浏览器兼容性**
   - `AsyncLocalStorage` 是 Node.js 特有 API
   - 该模块被设计为仅在 CLI/SDK 代码路径中导入
   - 如果意外导入到浏览器构建，会导致运行时错误

2. **上下文泄漏**
   - 虽然实现了边界保护，但 ALS 本身不是绝对安全的
   - 某些异步模式（如某些事件监听器）可能绕过 ALS

3. **Workload 值硬编码**
   - 当前仅支持 `'cron'` 作为有效值
   - 扩展需要同时修改 TypeScript 和 Python 验证代码

### 边界情况

1. **嵌套调用**
   ```typescript
   runWithWorkload('cron', () => {
     runWithWorkload(undefined, () => {
       getWorkload() // 返回 undefined，不是 'cron'
     })
   })
   ```
   每次调用都建立新边界，内部调用不会继承外部值。

2. **异步边界跨越**
   ```typescript
   await runWithWorkload('cron', async () => {
     await someAsyncOp()
     getWorkload() // 仍然返回 'cron'，ALS 保持上下文
   })
   ```

3. **回调函数**
   ```typescript
   runWithWorkload('cron', () => {
     setTimeout(() => {
       getWorkload() // 行为取决于 setTimeout 的调度时机
     }, 0)
   })
   ```
   如果回调在 `run` 返回后执行，ALS 上下文已退出。

### 改进建议

1. **Workload 值扩展**
   - 考虑支持更多 workload 类型（如 `'webhook'`, `'scheduled'`）
   - 建立统一的 workload 值注册机制

2. **调试支持**
   - 添加 `getWorkloadStack()` 函数，返回上下文链
   - 在调试模式下记录 workload 切换

3. **类型安全增强**
   ```typescript
   // 使用 branded type 区分已验证的 workload
   type ValidatedWorkload = string & { __brand: 'workload' }
   ```

4. **文档完善**
   - 添加更多使用示例
   - 说明与 `agentContext.ts` 的关系（代码注释提到使用相同模式）

5. **测试覆盖**
   - 添加单元测试验证上下文隔离行为
   - 测试嵌套调用和异步边界场景
