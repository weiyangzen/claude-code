# src/utils/configConstants.ts 深度研究文档

## 1. 场景与职责

`configConstants.ts` 是一个极简的常量定义文件，专门用于解决循环依赖问题。它将一些基础配置常量从 `config.ts` 中分离出来，确保这些常量可以在不引入 `config.ts` 及其复杂依赖树的情况下被其他模块使用。

### 核心职责
- **解耦常量定义**: 将通知渠道、编辑器模式等常量独立定义
- **避免循环依赖**: 作为无依赖的叶子节点模块，可被任意其他模块安全导入
- **类型安全**: 使用 `as const` 提供严格的 TypeScript 类型推断

## 2. 功能点目的

### 2.1 通知渠道常量

```typescript
export const NOTIFICATION_CHANNELS = [
  'auto',           // 自动选择（基于终端类型）
  'iterm2',         // iTerm2 原生通知
  'iterm2_with_bell', // iTerm2 通知 + 蜂鸣
  'terminal_bell',  // 终端蜂鸣
  'kitty',          // Kitty 终端通知
  'ghostty',        // Ghostty 终端通知
  'notifications_disabled', // 完全禁用通知
] as const
```

**使用场景**:
- 用户通过 `/config` 命令选择通知偏好
- 通知系统根据选择发送不同类型的通知
- 状态栏显示当前通知渠道

### 2.2 编辑器模式常量

```typescript
export const EDITOR_MODES = ['normal', 'vim'] as const
```

**注意**: `'emacs'` 模式已被弃用（在 `config.ts` 中通过 `EditorMode` 类型保留兼容）

**使用场景**:
- REPL 输入框的键盘处理
- 行编辑行为（如快捷键、光标移动）
- 帮助文本显示当前模式

### 2.3 队友模式常量

```typescript
export const TEAMMATE_MODES = ['auto', 'tmux', 'in-process'] as const
```

| 模式 | 说明 |
|------|------|
| `auto` | 根据上下文自动选择（默认） |
| `tmux` | 传统的基于 tmux 的队友进程 |
| `in-process` | 同进程内队友（实验性） |

**使用场景**:
- 多智能体任务执行时的队友进程创建
- 性能优化（in-process 避免进程间通信开销）
- 兼容性处理（tmux 在特定环境不可用）

## 3. 具体技术实现

### 3.1 设计原则

```typescript
// 文件顶部明确声明：禁止添加导入
// These constants are in a separate file to avoid circular dependency issues.
// Do NOT add imports to this file - it must remain dependency-free.
```

这是一个**叶子模块**（Leaf Module）：
- 无导入依赖
- 只有导出
- 可被依赖图中的任何节点安全引用

### 3.2 类型推导

使用 `as const` 断言实现严格的字面量类型：

```typescript
// 推导结果类型：readonly ['auto', 'iterm2', ...]
// 元素类型：'auto' | 'iterm2' | 'terminal_bell' | ...

// 使用示例
export type NotificationChannel = (typeof NOTIFICATION_CHANNELS)[number]
// 结果：type NotificationChannel = 'auto' | 'iterm2' | ...
```

## 4. 关键代码路径与文件引用

### 4.1 导出关系

```
configConstants.ts
├── NOTIFICATION_CHANNELS ──→ config.ts（重新导出）
│                          → src/utils/notifier.ts
│                          → src/components/Settings/Config.tsx
│                          → 其他使用通知渠道的模块
├── EDITOR_MODES ──────────→ config.ts（重新导出）
│                          → src/hooks/useInputBuffer.ts
│                          → src/commands/vim/vim.ts
│                          → 其他编辑器相关模块
└── TEAMMATE_MODES ────────→ src/utils/swarm/backends/teammateModeSnapshot.ts
                           → src/tools/shared/spawnMultiAgent.ts
```

### 4.2 在 config.ts 中的使用

```typescript
// config.ts 重新导出这些常量
export { EDITOR_MODES, NOTIFICATION_CHANNELS } from './configConstants.js'

// 类型定义
export type NotificationChannel = (typeof NOTIFICATION_CHANNELS)[number]

// 向后兼容的编辑器模式（包含已弃用的 'emacs'）
export type EditorMode = 'emacs' | (typeof EDITOR_MODES)[number]
```

### 4.3 使用示例

**通知渠道选择**:
```typescript
// src/utils/notifier.ts
import { NOTIFICATION_CHANNELS } from './configConstants.js'

function validateNotifChannel(channel: string): boolean {
  return NOTIFICATION_CHANNELS.includes(channel as NotificationChannel)
}
```

**编辑器模式切换**:
```typescript
// src/commands/vim/vim.ts
import { EDITOR_MODES } from '../utils/configConstants.js'

export function toggleEditorMode(current: EditorMode): EditorMode {
  const currentIndex = EDITOR_MODES.indexOf(current as 'normal' | 'vim')
  if (currentIndex === -1) return 'normal' // 处理已弃用的 'emacs'
  const nextIndex = (currentIndex + 1) % EDITOR_MODES.length
  return EDITOR_MODES[nextIndex]
}
```

## 5. 依赖与外部交互

### 5.1 依赖图位置

```
configConstants.ts（无依赖）
    ↑
    ├── config.ts
    ├── notifier.ts
    ├── useInputBuffer.ts
    ├── vim.ts
    ├── teammateModeSnapshot.ts
    └── ...（其他消费者）
```

### 5.2 零依赖保证

该文件**严格禁止**添加任何 `import` 语句。如果需要添加新的常量类型：

1. 确保新常量不依赖任何外部类型
2. 如果必须依赖，考虑将常量保留在 `config.ts` 中
3. 或者创建另一个独立的常量文件

## 6. 风险、边界与改进建议

### 6.1 风险分析

| 风险 | 可能性 | 影响 | 说明 |
|------|--------|------|------|
| 意外添加导入 | 中 | 高 | 可能引入循环依赖，破坏设计目的 |
| 常量值变更 | 低 | 高 | 已存储的配置值可能失效 |
| 类型不兼容 | 低 | 中 | 与其他系统的字符串枚举不匹配 |

### 6.2 边界情况

**向后兼容性**:
- `emacs` 编辑器模式已弃用，但仍在 `EditorMode` 类型中保留
- 迁移逻辑在 `config.ts` 中处理（自动将 'emacs' 视为 'normal'）

**配置验证**:
- 当前无运行时验证，依赖 TypeScript 编译时检查
- 手动编辑 `~/.claude.json` 可能存储无效值

### 6.3 改进建议

#### 短期
1. **添加 JSDoc 注释**: 为每个常量添加使用说明和示例
2. **运行时验证**: 提供验证函数（在另一个文件中，避免引入依赖）

#### 中期
3. **配置迁移**: 考虑将 `emacs` 自动迁移到 `normal` 的逻辑移到独立迁移脚本
4. **国际化**: 如果需要，准备显示名称的映射（注意：这会增加复杂性，可能违背本文件的简单性原则）

#### 长期
5. **代码生成**: 考虑从单一来源（如 JSON Schema）生成这些常量和类型
6. **文档同步**: 确保这些常量的变更自动同步到用户文档

### 6.4 维护指南

**添加新常量时**:
1. 确保新常量与现有风格一致（UPPER_SNAKE_CASE）
2. 更新本研究文档
3. 检查是否需要同步更新 `config.ts` 中的类型定义
4. 运行循环依赖检测工具验证

**修改现有常量时**:
1. 评估向后兼容性影响
2. 添加迁移逻辑（如果需要）
3. 更新所有相关文档
