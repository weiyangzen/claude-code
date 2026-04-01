# terminal.ts 研究文档

## 场景与职责

`terminal.ts` 是 Ink 框架的终端能力检测和输出控制核心模块。负责检测终端支持的各类高级功能（同步输出、进度报告、扩展键等），并提供统一的终端输出接口。

### 核心职责
1. **能力检测**: 检测终端支持的协议和功能
2. **进度报告**: OSC 9;4 进度条控制
3. **同步输出**: DEC 2026 同步更新序列管理
4. **终端输出**: 将帧差异写入终端
5. **终端识别**: XTVERSION 和 TERM_PROGRAM 综合识别

## 功能点目的

### 1. 进度报告检测 (`isProgressReportingAvailable`)
检测终端是否支持 OSC 9;4 进度报告：

**支持终端**:
- ConEmu (Windows) - 所有版本
- Ghostty 1.2.0+
- iTerm2 3.6.6+

**排除项**:
- Windows Terminal（将 OSC 9;4 解释为通知而非进度）
- 非 TTY 输出

### 2. 同步输出检测 (`isSynchronizedOutputSupported`)
检测 DEC 2026 同步更新支持，用于防止渲染闪烁：

**支持终端**:
- iTerm.app, WezTerm, WarpTerminal, ghostty
- VS Code, Alacritty, kitty, foot, contour
- Windows Terminal (WT_SESSION)
- VTE 0.68+ (GNOME Terminal, Tilix 等)

**特殊处理**:
- tmux 明确禁用（代理但不实现 DEC 2026）

### 3. 终端识别 (`isXtermJs`)
检测是否运行在 xterm.js 终端（VS Code, Cursor, Windsurf）：

```typescript
export function isXtermJs(): boolean {
  if (process.env.TERM_PROGRAM === 'vscode') return true
  return xtversionName?.startsWith('xterm.js') ?? false
}
```

结合环境变量和 XTVERSION 查询结果，支持 SSH 场景检测。

### 4. 扩展键检测 (`supportsExtendedKeys`)
检测终端是否正确处理扩展键报告（Kitty 键盘协议 + xterm modifyOtherKeys）：

**支持终端白名单**:
- iTerm.app, kitty, WezTerm, ghostty
- tmux（接受 modifyOtherKeys，不转发 Kitty 序列）
- windows-terminal

### 5. 光标上滚 Bug 检测 (`hasCursorUpViewportYankBug`)
检测 Windows conhost 的光标上滚导致的视口跳动问题：

```typescript
export function hasCursorUpViewportYankBug(): boolean {
  return process.platform === 'win32' || !!process.env.WT_SESSION
}
```

### 6. 终端输出 (`writeDiffToTerminal`)
将帧差异（Diff）序列化并写入终端：

```typescript
export function writeDiffToTerminal(
  terminal: Terminal,
  diff: Diff,
  skipSyncMarkers = false,
): void
```

**Patch 类型处理**:
- `stdout`: 直接输出内容
- `clear`: 擦除行序列
- `clearTerminal`: 清屏序列
- `cursorHide/Show`: 光标控制
- `cursorMove/To`: 光标定位
- `hyperlink`: OSC 8 超链接
- `styleStr`: 预序列化的样式转换

## 具体技术实现

### 能力检测实现
使用环境变量组合检测，优先级：
1. `TERM_PROGRAM` - 终端程序标识
2. `TERM` - 终端类型
3. 终端特定变量（`KITTY_WINDOW_ID`, `WT_SESSION` 等）
4. `VTE_VERSION` - VTE 库版本

### XTVERSION 异步检测
```typescript
let xtversionName: string | undefined

export function setXtversionName(name: string): void {
  if (xtversionName === undefined) xtversionName = name
}
```

`App.tsx` 在启用原始模式时发送 XTVERSION 查询，响应到达后调用 `setXtversionName()`。

### 同步输出包装
```typescript
const useSync = !skipSyncMarkers
let buffer = useSync ? BSU : ''  // Begin Synchronized Update

// ... 追加所有 patch ...

if (useSync) buffer += ESU  // End Synchronized Update
terminal.stdout.write(buffer)
```

BSU/ESU 序列包装整个帧输出，确保原子更新。

### 版本比较
使用 `semver.ts` 提供的比较函数：
```typescript
import { gte } from '../utils/semver.js'

if (process.env.TERM_PROGRAM === 'ghostty') {
  return gte(version.version, '1.2.0')
}
```

## 关键代码路径与文件引用

### 依赖
```typescript
import { coerce } from 'semver'
import { env } from '../utils/env.js'
import { gte } from '../utils/semver.js'
import { getClearTerminalSequence } from './clearTerminal.js'
import type { Diff } from './frame.js'
import { cursorMove, cursorTo, eraseLines } from './termio/csi.js'
import { BSU, ESU, HIDE_CURSOR, SHOW_CURSOR } from './termio/dec.js'
import { link } from './termio/osc.js'
```

### 调用方
- **`ink.tsx`**: 主渲染循环调用 `writeDiffToTerminal()`
- **`App.tsx`**: 调用 `setXtversionName()` 设置终端名称
- **各组件**: 使用能力检测函数决定是否启用高级功能

### 导出常量
```typescript
export const SYNC_OUTPUT_SUPPORTED = isSynchronizedOutputSupported()
```

模块加载时计算一次，供同步代码路径使用。

## 依赖与外部交互

| 依赖 | 用途 |
|------|------|
| `semver` | 版本号解析和比较 |
| `utils/env.ts` | 环境工具函数 |
| `utils/semver.ts` | Bun/Node 兼容的 semver 比较 |
| `clearTerminal.ts` | 清屏序列生成 |
| `frame.ts` | `Diff`, `Progress` 类型 |
| `termio/csi.ts` | CSI 光标控制序列 |
| `termio/dec.ts` | DEC 私有模式序列（BSU/ESU） |
| `termio/osc.ts` | OSC 超链接序列 |

## 风险、边界与改进建议

### 已知风险
1. **检测滞后**: 新终端支持需要手动添加检测逻辑
2. **SSH 转发**: 环境变量在 SSH 会话中可能丢失
3. **版本解析**: `semver.coerce()` 对某些版本格式可能失败

### 边界情况
1. **嵌套终端**: tmux 内运行其他终端模拟器时检测复杂
2. **版本边界**: 恰好在支持边界的版本可能检测错误
3. **并发检测**: 多个 Ink 实例同时查询可能冲突

### 改进建议
1. **查询缓存**: 缓存能力检测结果，避免重复计算
2. **用户覆盖**: 添加环境变量允许用户强制启用/禁用功能
3. **动态更新**: 支持运行时重新检测（终端热切换场景）
4. **标准检测**: 使用更标准化的终端能力查询（如 terminfo）
5. **遥测**: 收集检测失败案例，持续改进检测逻辑

### 相关标准
- DEC 2026 - Synchronized Output
- OSC 9;4 - iTerm2/Ghostty 进度报告
- XTVERSION - XTerm 版本查询
- Kitty 键盘协议
- xterm modifyOtherKeys
