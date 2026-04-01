# use-tab-status.ts 深入研究

## 场景与职责

`useTabStatus` 是 Ink 终端 UI 框架中用于设置标签页状态指示器的 Hook。它通过 OSC 21337 序列向终端标签侧边栏发送彩色圆点和状态文本，提供视觉状态反馈。

## 功能点目的

### 1. 标签页状态指示
- 在终端标签页显示彩色指示器
- 提供简短的状态文本描述
- 支持三种预设状态：空闲、忙碌、等待中

### 2. 终端兼容性
- 使用 OSC 21337 序列（Tab Status 扩展）
- 不支持的终端静默丢弃序列
- 自动包装以支持 tmux/screen 透传

### 3. 状态清理
- 支持传入 `null` 清除状态
- 组件卸载时自动清理
- 状态切换时清除旧状态

## 具体技术实现

### 状态类型定义

```typescript
export type TabStatusKind = 'idle' | 'busy' | 'waiting'

const TAB_STATUS_PRESETS: Record<TabStatusKind, {
  indicator: Color    // 指示器颜色
  status: string      // 状态文本
  statusColor: Color  // 状态文本颜色
}> = {
  idle: {
    indicator: rgb(0, 215, 95),    // 绿色
    status: 'Idle',
    statusColor: rgb(136, 136, 136),  // 灰色
  },
  busy: {
    indicator: rgb(255, 149, 0),   // 橙色
    status: 'Working…',
    statusColor: rgb(255, 149, 0),
  },
  waiting: {
    indicator: rgb(95, 135, 255),  // 蓝色
    status: 'Waiting',
    statusColor: rgb(95, 135, 255),
  },
}
```

### 核心实现逻辑

```typescript
export function useTabStatus(kind: TabStatusKind | null): void {
  const writeRaw = useContext(TerminalWriteContext)
  const prevKindRef = useRef<TabStatusKind | null>(null)

  useEffect(() => {
    // 从非 null 切换到 null 时清除状态
    if (kind === null) {
      if (prevKindRef.current !== null && writeRaw && supportsTabStatus()) {
        writeRaw(wrapForMultiplexer(CLEAR_TAB_STATUS))
      }
      prevKindRef.current = null
      return
    }

    prevKindRef.current = kind
    if (!writeRaw || !supportsTabStatus()) return
    writeRaw(wrapForMultiplexer(tabStatus(TAB_STATUS_PRESETS[kind])))
  }, [kind, writeRaw])
}
```

### 颜色辅助函数

```typescript
const rgb = (r: number, g: number, b: number): Color => ({
  type: 'rgb',
  r, g, b,
})
```

## 关键代码路径与文件引用

### 依赖文件

| 文件路径 | 作用 |
|---------|------|
| `src/ink/termio/osc.ts` | 提供 OSC 序列生成和 tabStatus 函数 |
| `src/ink/useTerminalNotification.ts` | 提供 TerminalWriteContext |
| `src/ink/termio/types.ts` | 定义 Color 类型 |

### OSC 相关函数

```typescript
// src/ink/termio/osc.ts

// OSC 命令号
export const OSC = {
  TAB_STATUS: 21337,
  // ...
}

// 检查是否支持 Tab Status
export function supportsTabStatus(): boolean {
  return process.env.USER_TYPE === 'ant'
}

// 生成 Tab Status 序列
export function tabStatus(fields: TabStatusAction): string {
  const parts: string[] = []
  const rgb = (c: Color) =>
    c.type === 'rgb'
      ? `#${[c.r, c.g, c.b].map(n => n.toString(16).padStart(2, '0')).join('')}`
      : ''
  if ('indicator' in fields)
    parts.push(`indicator=${fields.indicator ? rgb(fields.indicator) : ''}`)
  if ('status' in fields)
    parts.push(`status=${fields.status?.replaceAll('\\', '\\\\').replaceAll(';', '\\;') ?? ''}`)
  if ('statusColor' in fields)
    parts.push(`status-color=${fields.statusColor ? rgb(fields.statusColor) : ''}`)
  return osc(OSC.TAB_STATUS, parts.join(';'))
}

// 清除 Tab Status
export const CLEAR_TAB_STATUS = osc(
  OSC.TAB_STATUS,
  'indicator=;status=;status-color=',
)

// 多路复用器包装（tmux/screen）
export function wrapForMultiplexer(sequence: string): string {
  if (process.env['TMUX']) {
    const escaped = sequence.replaceAll('\x1b', '\x1b\x1b')
    return `\x1bPtmux;${escaped}\x1b\\`
  }
  if (process.env['STY']) {
    return `\x1bP${sequence}\x1b\\`
  }
  return sequence
}
```

### 序列格式

OSC 21337 序列格式：
```
ESC ] 21337 ; key=value;key=value BEL
```

支持的字段：
- `indicator`: 指示器颜色（#RRGGBB 格式）
- `status`: 状态文本（`;` 和 `\` 需要转义）
- `status-color`: 状态文本颜色

## 依赖与外部交互

### 与 TerminalWriteContext 的交互

```typescript
const writeRaw = useContext(TerminalWriteContext)
```

- `writeRaw` 用于向终端写入原始序列
- 由 Ink 主组件通过 Provider 提供
- 确保序列在正确的时机写入

### 与终端的交互

1. **序列发送**：
   - 通过 `writeRaw` 发送 OSC 序列
   - 自动包装以支持 tmux/screen

2. **终端支持检测**：
   - 目前仅当 `USER_TYPE === 'ant'` 时启用
   - 这是临时限制，等待规范稳定

3. **清理**：
   - 进程退出时由 `ink.tsx` 的卸载路径处理
   - 状态切换时发送 CLEAR_TAB_STATUS

### 使用示例

```typescript
import { useTabStatus } from 'ink'

const TaskRunner = ({ isRunning }) => {
  // 根据任务状态设置标签页状态
  useTabStatus(isRunning ? 'busy' : 'idle')
  
  return <Text>Task {isRunning ? 'running' : 'idle'}</Text>
}

// 或条件性禁用
const OptionalStatus = ({ showStatus, isBusy }) => {
  useTabStatus(showStatus ? (isBusy ? 'busy' : 'idle') : null)
  
  return <Text>Content</Text>
}
```

## 风险、边界与改进建议

### 潜在风险

1. **规范不稳定**：
   - OSC 21337 是自定义扩展，规范可能变化
   - 目前限制为 Ant 内部使用

2. **终端兼容性**：
   - 需要终端显式支持 OSC 21337
   - 不支持的终端静默丢弃序列（无害但无效果）

3. **tmux 配置依赖**：
   - 需要 `allow-passthrough on` 才能透传
   - 默认关闭，用户需要手动配置

### 边界情况

1. **writeRaw 未提供**：
   - Hook 静默返回，不执行任何操作
   - 不会抛出错误

2. **不支持 Tab Status**：
   - `supportsTabStatus()` 返回 false
   - 不发送序列

3. **快速状态切换**：
   - 每次切换都发送新序列
   - 可能产生闪烁

4. **特殊字符**：
   - 状态文本中的 `;` 和 `\` 自动转义
   - 其他特殊字符可能需要额外处理

### 改进建议

1. **扩展终端支持**：
   ```typescript
   export function supportsTabStatus(): boolean {
     // 检测支持的终端
     const supportedTerminals = ['ant', 'iterm2', 'kitty', 'ghostty']
     return supportedTerminals.some(t => 
       process.env.TERM_PROGRAM?.toLowerCase().includes(t)
     )
   }
   ```

2. **添加防抖**：
   ```typescript
   useEffect(() => {
     const timer = setTimeout(() => {
       // 实际发送逻辑
     }, 100)
     return () => clearTimeout(timer)
   }, [kind])
   ```

3. **支持自定义状态**：
   ```typescript
   useTabStatus({
     indicator: rgb(255, 0, 0),
     status: 'Custom',
     statusColor: rgb(255, 255, 255)
   })
   ```

4. **添加状态队列**：
   ```typescript
   // 支持优先级状态
   useTabStatus('busy', { priority: 'high' })
   useTabStatus('waiting', { priority: 'low' })  // 不会覆盖 high
   ```

5. **暴露检测函数**：
   ```typescript
   export function useTabStatusSupport(): boolean {
     return supportsTabStatus()
   }
   ```

### 测试建议

1. 测试三种预设状态的显示
2. 测试 null 值清除状态
3. 测试 tmux/screen 透传
4. 测试特殊字符转义
5. 测试快速状态切换
6. 测试不支持的终端（应静默失败）
