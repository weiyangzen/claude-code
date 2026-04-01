# staticRender.tsx 深度研究

## 场景与职责

`staticRender.tsx` 提供**将 React 组件渲染为字符串的工具**，支持 ANSI 转义码（终端输出）和纯文本两种格式。

**核心职责：**
1. 渲染 React 节点到字符串
2. 支持 ANSI 转义码输出
3. 支持纯文本输出（去除 ANSI）
4. 处理 Ink 在非 TTY 输出下的多帧问题

**应用场景：**
- 计划命令输出到文件
- 上下文命令导出
- 非交互式输出

---

## 功能点目的

### 1. 渲染并退出组件
```typescript
function RenderOnceAndExit({ children }: { children: React.ReactNode }): React.ReactNode
```

**功能：**
- 使用 `useLayoutEffect` 确保 React 提交阶段完成后退出
- 使用 `setTimeout(..., 0)` 延迟退出
- 比 `process.nextTick()` 更可靠（React 19 异步渲染）

### 2. 首帧内容提取
```typescript
const SYNC_START = '\x1B[?2026h'
const SYNC_END = '\x1B[?2026l'

function extractFirstFrame(output: string): string
```

**背景：**
- Ink 在非 TTY 输出下输出多帧
- 每帧包裹在 DEC 同步更新序列中
- 只需提取第一帧内容避免重复

### 3. ANSI 字符串渲染
```typescript
export function renderToAnsiString(
  node: React.ReactNode,
  columns?: number
): Promise<string>
```

**流程：**
1. 创建 `PassThrough` 流捕获输出
2. 可选设置 `columns` 属性控制宽度
3. 渲染 `RenderOnceAndExit` 包裹的组件
4. 等待退出
5. 提取并返回第一帧内容

### 4. 纯文本渲染
```typescript
export async function renderToString(
  node: React.ReactNode,
  columns?: number
): Promise<string>
```

**功能：**
- 调用 `renderToAnsiString`
- 使用 `strip-ansi` 去除 ANSI 转义码

---

## 具体技术实现

### RenderOnceAndExit 实现
```typescript
function RenderOnceAndExit({ children }: { children: React.ReactNode }): React.ReactNode {
  const { exit } = useApp()
  
  useLayoutEffect(() => {
    const timer = setTimeout(exit, 0)
    return () => clearTimeout(timer)
  }, [exit])
  
  return <>{children}</>
}
```

### ANSI 渲染流程
```typescript
export function renderToAnsiString(node: React.ReactNode, columns?: number): Promise<string> {
  return new Promise(async resolve => {
    let output = ''
    
    const stream = new PassThrough()
    if (columns !== undefined) {
      (stream as unknown as { columns: number }).columns = columns
    }
    
    stream.on('data', chunk => {
      output += chunk.toString()
    })
    
    const instance = await render(
      <RenderOnceAndExit>{node}</RenderOnceAndExit>,
      { stdout: stream as unknown as NodeJS.WriteStream, patchConsole: false }
    )
    
    await instance.waitUntilExit()
    resolve(extractFirstFrame(output))
  })
}
```

---

## 关键代码路径与文件引用

### 核心导出
| 导出 | 用途 |
|------|------|
| `renderToAnsiString` | 渲染为 ANSI 字符串 |
| `renderToString` | 渲染为纯文本 |

### 依赖模块
| 模块 | 用途 |
|------|------|
| `react` | React 核心 |
| `stream` | `PassThrough` |
| `strip-ansi` | ANSI 去除 |
| `../ink.js` | `render`, `useApp` |

### 调用方
| 文件 | 用途 |
|------|------|
| `src/commands/plan/plan.tsx` | 计划命令 |
| `src/commands/context/context.tsx` | 上下文命令 |
| `src/utils/exportRenderer.tsx` | 导出渲染 |

---

## 依赖与外部交互

### 外部依赖
| 模块 | 用途 |
|------|------|
| `react` | React 运行时 |
| `stream` | 流操作 |
| `strip-ansi` | ANSI 去除 |

### 内部依赖
| 模块 | 用途 |
|------|------|
| `../ink.js` | Ink 渲染 |

---

## 风险、边界与改进建议

### 已知风险

1. **异步渲染**
   - React 19 异步渲染可能导致时序问题
   - 缓解：使用 `useLayoutEffect`

2. **多帧问题**
   - Ink 非 TTY 输出产生多帧
   - 依赖 DEC 同步序列提取第一帧

3. **内存泄漏**
   - 流事件监听器未明确移除
   - 依赖垃圾回收

### 边界情况

| 场景 | 处理 |
|------|------|
| 无 DEC 序列 | `extractFirstFrame` 返回原字符串 |
| 空组件 | 返回空字符串 |
| columns 未设置 | Ink 使用默认 80 列 |
| 渲染错误 | 由 Ink 处理，Promise 可能 reject |

### 改进建议

1. **错误处理**
   - 添加渲染错误捕获
   - 提供有意义的错误信息

2. **性能优化**
   - 复用 PassThrough 流实例
   - 添加渲染缓存

3. **功能扩展**
   - 支持流式输出
   - 添加渲染超时

4. **测试覆盖**
   - 添加单元测试
   - 测试各种组件类型

5. **Ink 升级**
   - 关注 Ink 更新解决多帧问题
   - 评估官方静态渲染 API
