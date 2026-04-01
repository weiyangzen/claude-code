# supports-hyperlinks.ts 研究文档

## 场景与职责

`supports-hyperlinks.ts` 负责检测当前终端是否支持 OSC 8 超链接协议。OSC 8 允许在终端中创建可点击的链接，格式为 `ESC]8;;URL ESC\` 或 `ESC]8;;URL BEL`。

### 核心职责
1. **终端能力检测**：判断 stdout 是否支持超链接渲染
2. **扩展终端支持**：补充 `supports-hyperlinks` 库未覆盖的终端类型
3. **tmux 兼容**：通过 `LC_TERMINAL` 检测 tmux 内的原始终端

## 功能点目的

### 1. 基础检测
使用 `supports-hyperlinks` npm 包进行基础检测，该库检查：
- `TERM_PROGRAM` 环境变量
- 终端模拟器特定标志

### 2. 扩展终端白名单
```typescript
export const ADDITIONAL_HYPERLINK_TERMINALS = [
  'ghostty',
  'Hyper', 
  'kitty',
  'alacritty',
  'iTerm.app',
  'iTerm2',
]
```

这些终端明确支持 OSC 8 但可能未被上游库识别。

### 3. 多层级检测策略
检测顺序（优先级从高到低）：
1. `supports-hyperlinks` 库检测结果
2. `TERM_PROGRAM` 白名单匹配
3. `LC_TERMINAL` 白名单匹配（tmux 内保留）
4. `TERM` 包含 "kitty"

## 具体技术实现

### 检测函数
```typescript
export function supportsHyperlinks(options?: SupportsHyperlinksOptions): boolean
```

支持测试注入：
- `env`: 模拟环境变量
- `stdoutSupported`: 强制指定 stdout 支持状态

### tmux 特殊处理
```typescript
const lcTerminal = env['LC_TERMINAL']
if (lcTerminal && ADDITIONAL_HYPERLINK_TERMINALS.includes(lcTerminal)) {
  return true
}
```

在 tmux 会话中，`TERM_PROGRAM` 会被覆盖为 `tmux`，但 `LC_TERMINAL` 保留原始终端信息。

### Kitty 特殊检测
```typescript
if (term?.includes('kitty')) {
  return true
}
```

Kitty 设置 `TERM=xterm-kitty`，通过字符串包含检测捕获。

## 关键代码路径与文件引用

### 调用方
- **`termio/osc.ts`**: `link()` 函数在生成超链接前调用检测
- **组件层**: `Link` 组件决定是否渲染可点击链接

### 依赖
```typescript
import supportsHyperlinksLib from 'supports-hyperlinks'
```

## 依赖与外部交互

| 依赖 | 用途 |
|------|------|
| `supports-hyperlinks` | 基础终端超链接支持检测 |
| `process.env` | 环境变量读取 |

## 风险、边界与改进建议

### 已知风险
1. **检测滞后**: 新终端支持 OSC 8 需要手动添加到白名单
2. **SSH 转发**: 通过 SSH 连接时环境变量可能丢失
3. **嵌套终端**: 在终端复用器（tmux/screen）中检测复杂

### 边界情况
1. **CI 环境**: 大多数 CI 环境不支持超链接，会静默降级为纯文本
2. **Windows 终端**: 旧版 Windows 控制台不支持 OSC 8
3. **管道输出**: 重定向到文件时超链接序列会污染输出

### 改进建议
1. **动态检测**: 实现基于查询-响应的实时检测（如 XTVERSION）
2. **用户覆盖**: 添加 `FORCE_HYPERLINKS` 环境变量允许用户强制启用
3. **缓存结果**: 检测结果在会话期间缓存，避免重复计算
4. **扩展白名单**: 定期同步上游 `supports-hyperlinks` 更新

### 相关标准
- [OSC 8 规范](https://gist.github.com/egmontkob/eb114294ef5d1f3b6a2dbb0ef2007c83)
- Hyperlinks in Terminal Emulators
