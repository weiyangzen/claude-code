# osc.ts 研究报告

## 场景与职责

`osc.ts` 实现了 **OSC (Operating System Command)** 序列的完整支持。OSC 是终端与操作系统/宿主环境通信的机制，格式为 `ESC ] <命令> ; <参数> <终止符>`，用于设置窗口标题、操作剪贴板、显示超链接、通知等。

该文件的核心职责：
1. **定义 OSC 命令号常量** —— 标题设置、颜色设置、剪贴板操作、超链接等
2. **生成 OSC 序列** —— 提供 `osc()` 函数和便捷函数
3. **解析 OSC 序列** —— 将 OSC 内容解析为语义动作
4. **剪贴板集成** —— 多路径剪贴板写入（原生工具、tmux、OSC 52）
5. **多路复用器透传** —— 支持 tmux/screen 的 DCS 透传包装

## 功能点目的

### 1. OSC 命令号定义

```typescript
export const OSC = {
  SET_TITLE_AND_ICON: 0,   // 设置窗口标题和图标
  SET_ICON: 1,             // 设置图标
  SET_TITLE: 2,            // 设置窗口标题
  SET_COLOR: 4,            // 设置颜色（已弃用）
  SET_CWD: 7,              // 设置当前工作目录（iTerm2）
  HYPERLINK: 8,            // 超链接（OSC 8）
  ITERM2: 9,               // iTerm2 专有序列
  SET_FG_COLOR: 10,        // 设置前景色
  SET_BG_COLOR: 11,        // 设置背景色
  SET_CURSOR_COLOR: 12,    // 设置光标颜色
  CLIPBOARD: 52,           // 剪贴板操作
  KITTY: 99,               // Kitty 通知协议
  RESET_COLOR: 104,        // 重置颜色
  RESET_FG_COLOR: 110,     // 重置前景色
  RESET_BG_COLOR: 111,     // 重置背景色
  RESET_CURSOR_COLOR: 112, // 重置光标颜色
  SEMANTIC_PROMPT: 133,    // 语义提示（shell 集成）
  GHOSTTY: 777,            // Ghostty 通知协议
  TAB_STATUS: 21337,       // 标签状态扩展（Ant 内部）
} as const
```

### 2. 序列生成与终止符

```typescript
// OSC 序列前缀: ESC ]
export const OSC_PREFIX = ESC + String.fromCharCode(ESC_TYPE.OSC)

// 字符串终止符: ESC \
export const ST = ESC + '\\'

// 生成 OSC 序列
export function osc(...parts: (string | number)[]): string {
  const terminator = env.terminal === 'kitty' ? ST : BEL
  return `${OSC_PREFIX}${parts.join(SEP)}${terminator}`
}
```

**终止符选择**：
- **Kitty**：使用 ST (`ESC \`)，避免响铃
- **其他**：使用 BEL (`\x07`)，兼容性更好

### 3. 多路复用器透传

```typescript
export function wrapForMultiplexer(sequence: string): string {
  if (process.env['TMUX']) {
    // tmux: ESC P tmux ; <escaped> ESC \
    const escaped = sequence.replaceAll('\x1b', '\x1b\x1b')
    return `\x1bPtmux;${escaped}\x1b\\`
  }
  if (process.env['STY']) {
    // GNU screen: ESC P <sequence> ESC \
    return `\x1bP${sequence}\x1b\\`
  }
  return sequence
}
```

### 4. 剪贴板操作

```typescript
export type ClipboardPath = 'native' | 'tmux-buffer' | 'osc52'

export function getClipboardPath(): ClipboardPath {
  const nativeAvailable =
    process.platform === 'darwin' && !process.env['SSH_CONNECTION']
  if (nativeAvailable) return 'native'
  if (process.env['TMUX']) return 'tmux-buffer'
  return 'osc52'
}

export async function setClipboard(text: string): Promise<string>
```

**剪贴板写入策略**：
1. **本地 macOS**：`pbcopy`（原生工具）
2. **tmux 环境**：`tmux load-buffer -w -`（tmux 3.2+ 支持 `-w` 转发到外层终端）
3. **其他**：OSC 52 序列写入 stdout

**iTerm2 特殊处理**：tmux 的 OSC 52 转发会导致 iTerm2 崩溃（issue #22432），对 iTerm2 禁用 `-w` 标志。

### 5. 超链接支持（OSC 8）

```typescript
export function link(url: string, params?: Record<string, string>): string {
  if (!url) return LINK_END
  const p = { id: osc8Id(url), ...params }
  const paramStr = Object.entries(p)
    .map(([k, v]) => `${k}=${v}`)
    .join(':')
  return osc(OSC.HYPERLINK, paramStr, url)
}

export const LINK_END = osc(OSC.HYPERLINK, '', '')
```

**自动 ID 生成**：为相同 URL 生成一致 ID，使终端能将跨行的同一链接合并。

### 6. iTerm2 专有功能

```typescript
export const ITERM2 = {
  NOTIFY: 0,       // 通知
  BADGE: 2,        // 徽章
  PROGRESS: 4,     // 进度条
} as const

export const PROGRESS = {
  CLEAR: 0,
  SET: 1,
  ERROR: 2,
  INDETERMINATE: 3,
} as const

export const CLEAR_ITERM2_PROGRESS = `${OSC_PREFIX}${OSC.ITERM2};${ITERM2.PROGRESS};${PROGRESS.CLEAR};${BEL}`
```

### 7. 标签状态（OSC 21337）

```typescript
export function supportsTabStatus(): boolean {
  return process.env.USER_TYPE === 'ant'
}

export function tabStatus(fields: TabStatusAction): string {
  // 生成: ESC ] 21337 ; indicator=#rrggbb;status=text;status-color=#rrggbb BEL
}
```

**Ant 内部扩展**：仅当 `USER_TYPE=ant` 时启用，用于在终端标签页显示状态指示器。

### 8. OSC 解析

```typescript
export function parseOSC(content: string): Action | null {
  const semicolonIdx = content.indexOf(';')
  const command = semicolonIdx >= 0 ? content.slice(0, semicolonIdx) : content
  const data = semicolonIdx >= 0 ? content.slice(semicolonIdx + 1) : ''
  const commandNum = parseInt(command, 10)

  // 标题设置
  if (commandNum === OSC.SET_TITLE_AND_ICON) {
    return { type: 'title', action: { type: 'both', title: data } }
  }
  // 超链接
  if (commandNum === OSC.HYPERLINK) {
    return parseHyperlink(data)
  }
  // 标签状态
  if (commandNum === OSC.TAB_STATUS) {
    return { type: 'tabStatus', action: parseTabStatus(data) }
  }
  // ...
}
```

### 9. 颜色解析

```typescript
export function parseOscColor(spec: string): Color | null {
  // 支持格式:
  // #RRGGBB
  // rgb:R/G/B (1-4 位十六进制，缩放到 8 位)
}
```

## 关键代码路径与文件引用

### 依赖

| 被导入 | 来源 | 用途 |
|--------|------|------|
| `Buffer` | `buffer` | Base64 编码 |
| `env` | `../../utils/env.js` | 检测 Kitty 终端 |
| `execFileNoThrow` | `../../utils/execFileNoThrow.js` | 执行剪贴板工具 |
| `BEL`, `ESC`, `ESC_TYPE`, `SEP` | `./ansi.js` | 序列构建 |
| `Action`, `Color`, `TabStatusAction` | `./types.js` | 返回类型 |

### 被导入方

| 导入方 | 导入内容 | 用途 |
|--------|----------|------|
| `parser.ts` | `parseOSC` | 解析 OSC 序列 |
| `terminal.ts` | `link` | 超链接输出 |
| `use-terminal-title.ts` | `OSC`, `osc` | 设置终端标题 |
| `use-tab-status.ts` | `CLEAR_TAB_STATUS`, `supportsTabStatus`, `tabStatus`, `wrapForMultiplexer` | 标签状态管理 |
| `copy.tsx` | `setClipboard`, `getClipboardPath` | 剪贴板操作 |

### 导出内容

```typescript
// 常量
export const OSC_PREFIX, ST
export const OSC, ITERM2, PROGRESS
export const CLEAR_ITERM2_PROGRESS, CLEAR_TERMINAL_TITLE, CLEAR_TAB_STATUS
export const LINK_END

// 类型
export type ClipboardPath

// 函数
export function osc(...parts: (string | number)[]): string
export function wrapForMultiplexer(sequence: string): string
export function getClipboardPath(): ClipboardPath
export function setClipboard(text: string): Promise<string>
export function parseOSC(content: string): Action | null
export function parseOscColor(spec: string): Color | null
export function link(url: string, params?: Record<string, string>): string
export function supportsTabStatus(): boolean
export function tabStatus(fields: TabStatusAction): string

// 内部测试导出
export function _resetLinuxCopyCache(): void
```

## 依赖与外部交互

### 内部依赖

- `ansi.ts`：基础常量
- `types.ts`：类型定义
- `env.ts`：终端检测
- `execFileNoThrow.ts`：子进程执行

### 外部使用场景

1. **终端标题**：`useTerminalTitle` 使用 `osc(OSC.SET_TITLE_AND_ICON, title)`
2. **剪贴板复制**：`copy.tsx` 使用 `setClipboard()` 实现多路径复制
3. **超链接**：`terminal.ts` 使用 `link()` 包装超链接输出
4. **标签状态**：`useTabStatus` 使用 `tabStatus()` 和 `wrapForMultiplexer()`
5. **OSC 解析**：`parser.ts` 调用 `parseOSC()` 处理 OSC 序列

### 外部工具依赖

| 平台 | 工具 | 用途 |
|------|------|------|
| macOS | `pbcopy` | 系统剪贴板 |
| Linux (Wayland) | `wl-copy` | Wayland 剪贴板 |
| Linux (X11) | `xclip`, `xsel` | X11 剪贴板 |
| Windows | `clip.exe` | 系统剪贴板 |
| tmux | `tmux load-buffer` | tmux 缓冲区 |

## 风险、边界与改进建议

### 边界情况

1. **终止符选择**：Kitty 使用 ST 避免响铃，但某些旧终端可能不支持 ST
2. **剪贴板大小限制**：OSC 52 有终端特定的剪贴板大小限制（通常几 KB 到几 MB）
3. **tmux 透传**：需要 `allow-passthrough on` 配置，否则序列被静默丢弃
4. **SSH 检测**：使用 `SSH_CONNECTION` 而非 `SSH_TTY`，因为 tmux pane 继承 `SSH_TTY`

### 风险

1. **竞态条件**：`setClipboard` 中 `pbcopy` 在 `tmux load-buffer` 之前启动，避免焦点切换导致的复制失败
2. **iTerm2 崩溃**：tmux 的 OSC 52 转发会导致 iTerm2 崩溃，已特殊处理
3. **Linux 工具探测**：首次调用时顺序探测 `wl-copy` → `xclip` → `xsel`，可能有延迟

### 改进建议

1. **添加更多 OSC 命令**：
   - OSC 9（iTerm2 通知）完整支持
   - OSC 133（语义提示）解析支持
   - OSC 777（Ghostty 通知）

2. **剪贴板改进**：
   - 添加 Windows WSL 支持（使用 `clip.exe` 或 Windows 原生）
   - 添加剪贴板大小检查和分块传输
   - 支持更多 Linux 剪贴板管理器（`copyq`, `gpaste`）

3. **错误处理**：
   - `setClipboard` 返回更详细的错误信息
   - 添加剪贴板操作超时处理

4. **性能优化**：
   - 缓存 `getClipboardPath()` 结果
   - 预生成常用 OSC 序列

5. **文档改进**：
   - 添加终端兼容性矩阵
   - 添加 OSC 52 大小限制说明

### 测试建议

- 测试各种 OSC 序列生成
- 测试剪贴板多路径选择逻辑
- 测试 `wrapForMultiplexer` 在各种环境下的行为
- 测试 `parseOscColor` 各种颜色格式
- 测试超链接 ID 生成一致性
- 验证与 tmux、screen 的兼容性
