# useTerminalNotification.ts 研究文档

## 场景与职责

`useTerminalNotification.ts` 提供 React Hook 接口，用于向终端发送各类通知。支持 iTerm2、Kitty、Ghostty 等终端的原生通知协议，以及 OSC 9;4 进度报告。

### 核心职责
1. **终端通知**: 发送桌面级通知到终端
2. **进度报告**: 更新终端任务栏/标签页进度指示器
3. **响铃**: 发送 BEL 字符触发终端响铃
4. **多路复用器兼容**: 自动处理 tmux/screen 的序列透传

## 功能点目的

### 1. iTerm2 通知 (`notifyITerm2`)
使用 OSC 9;0 协议发送通知：
```typescript
notifyITerm2({ message: string; title?: string })
```

格式：`ESC]9;

Title:
Message BEL`

### 2. Kitty 通知 (`notifyKitty`)
使用 OSC 99 协议发送富通知：
```typescript
notifyKitty({ message: string; title: string; id: number })
```

多包格式：
- `i={id}:d=0:p=title` - 标题包
- `i={id}:p=body` - 内容包  
- `i={id}:d=1:a=focus` - 完成包（请求焦点）

### 3. Ghostty 通知 (`notifyGhostty`)
使用 OSC 777 协议：
```typescript
notifyGhostty({ message: string; title: string })
```

格式：`ESC]777;notify;title;message BEL`

### 4. 终端响铃 (`notifyBell`)
发送原始 BEL 字符（`\x07`）：
- 在 tmux 内触发 tmux 的 bell-action（窗口标记）
- 不包装为 DCS，保持 tmux 的默认行为

### 5. 进度报告 (`progress`)
使用 OSC 9;4 协议更新进度条：
```typescript
progress(state: 'running' | 'completed' | 'error' | 'indeterminate' | null, percentage?: number)
```

状态映射：
| 状态 | 操作码 | 说明 |
|------|--------|------|
| `null` / `completed` | 0 (CLEAR) | 清除进度 |
| `running` | 1 (SET) | 设置具体百分比 |
| `error` | 2 (ERROR) | 错误状态 |
| `indeterminate` | 3 (INDETERMINATE) | 不确定进度 |

## 具体技术实现

### React Context 依赖
```typescript
const TerminalWriteContext = createContext<WriteRaw | null>(null)
```

通过 Context 获取原始写入函数，确保：
- 在 Ink 应用树内使用
- 自动应用多路复用器包装

### 多路复用器处理
```typescript
writeRaw(wrapForMultiplexer(osc(OSC.ITERM2, `\n\n${displayString}`)))
```

`wrapForMultiplexer()` 自动检测并处理：
- **tmux**: `ESC P tmux ; <escaped> ESC \`
- **screen**: `ESC P <sequence> ESC \`

### 进度值处理
```typescript
const pct = Math.max(0, Math.min(100, Math.round(percentage ?? 0)))
```

确保百分比值在 0-100 范围内，并四舍五入为整数。

### 能力检测
```typescript
if (!isProgressReportingAvailable()) {
  return
}
```

进度报告前检测终端支持，避免发送不支持的序列。

## 关键代码路径与文件引用

### 依赖
```typescript
import { createContext, useCallback, useContext, useMemo } from 'react'
import { isProgressReportingAvailable, type Progress } from './terminal.js'
import { BEL } from './termio/ansi.js'
import { ITERM2, OSC, osc, PROGRESS, wrapForMultiplexer } from './termio/osc.js'
```

### 调用方
- **组件层**: 任务完成、错误、长时间运行操作时发送通知
- **进度组件**: 实时更新任务进度

### 使用示例
```typescript
function TaskComponent() {
  const { notifyITerm2, progress } = useTerminalNotification()
  
  useEffect(() => {
    notifyITerm2({ title: 'Task Complete', message: 'Build finished successfully' })
  }, [])
  
  useEffect(() => {
    progress('running', 50)  // 50% 进度
    return () => progress(null)  // 清理
  }, [])
}
```

## 依赖与外部交互

| 依赖 | 用途 |
|------|------|
| `react` | Context 和 Hook API |
| `terminal.ts` | 进度报告能力检测 |
| `termio/ansi.ts` | BEL 常量 |
| `termio/osc.ts` | OSC 序列构造和多路复用器包装 |

### 外部交互
- **终端通知系统**: iTerm2/Kitty/Ghostty 原生通知
- **tmux**: 通过 DCS 透传序列
- **操作系统**: 桌面通知（通过终端代理）

## 风险、边界与改进建议

### 已知风险
1. **权限**: 某些终端需要用户授权才能显示通知
2. **干扰**: 频繁通知可能干扰用户
3. **兼容性**: 不支持通知的终端静默失败

### 边界情况
1. **tmux 透传**: 需要 `allow-passthrough on` 配置
2. **SSH 会话**: 通知显示在本地终端，而非远程服务器
3. **嵌套终端**: 多层 tmux/screen 可能阻断序列

### 改进建议
1. **批量通知**: 支持批量发送多个通知减少序列开销
2. **优先级**: 添加通知优先级控制
3. **回退**: 在不支持原生通知时回退到控制台消息
4. **防抖**: 对高频进度更新进行防抖
5. **持久化**: 支持跨会话的通知历史

### 相关标准
- OSC 9 - iTerm2 通知协议
- OSC 99 - Kitty 通知协议
- OSC 777 - Ghostty 通知协议
- OSC 9;4 - iTerm2/Ghostty 进度报告
