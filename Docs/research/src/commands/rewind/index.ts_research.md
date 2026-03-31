# src/commands/rewind/index.ts 研究文档

## 场景与职责

本文件是 `/rewind`（回退）命令的入口定义文件，负责声明命令的元数据、类型和懒加载配置。它是 Claude Code 命令系统中「本地命令（Local Command）」的典型示例，通过简洁的配置对象定义了用户如何通过斜杠命令触发消息选择器（Message Selector）功能。

**核心职责：**
- 定义 `/rewind` 命令的基本信息（名称、描述、别名）
- 声明命令类型为 `local`（本地执行，非 prompt 类型）
- 配置懒加载，避免启动时加载实现代码
- 标记不支持非交互式会话

## 功能点目的

### 1. 命令元数据定义

```typescript
const rewind = {
  description: `Restore the code and/or conversation to a previous point`,
  name: 'rewind',
  aliases: ['checkpoint'],
  argumentHint: '',
  type: 'local',
  supportsNonInteractive: false,
  load: () => import('./rewind.js'),
} satisfies Command
```

| 字段 | 值 | 说明 |
|------|-----|------|
| `name` | `'rewind'` | 主命令名，用户输入 `/rewind` 触发 |
| `aliases` | `['checkpoint']` | 别名 `/checkpoint` 同样有效 |
| `description` | 恢复代码和/或对话到之前的状态 | 帮助文本和类型提示中显示 |
| `type` | `'local'` | 本地命令类型，直接执行而非发送给模型 |
| `supportsNonInteractive` | `false` | 不支持非交互式模式（需要 TUI） |
| `load` | 动态导入 | 懒加载实际实现代码 |

### 2. 懒加载机制

使用 `load: () => import('./rewind.js')` 实现代码分割，确保：
- 应用启动时不加载 `rewind.ts` 的实现代码
- 首次执行 `/rewind` 时才动态导入
- 减少启动时间和内存占用

## 具体技术实现

### 类型系统

文件使用 TypeScript 的 `satisfies` 关键字确保配置对象符合 `Command` 类型：

```typescript
import type { Command } from '../../commands.js'
// ...
} satisfies Command
```

这提供了：
- 类型检查：确保所有必需字段存在
- 智能提示：IDE 可提供自动补全
- 类型安全：修改时捕获类型错误

### 命令注册流程

1. **导入阶段**（`src/commands.ts` 第 137 行）：
   ```typescript
   import rewind from './commands/rewind/index.js'
   ```

2. **注册阶段**（`src/commands.ts` 第 310 行）：
   ```typescript
   rewind,
   ```
   将命令加入 `COMMANDS` 数组，成为可用命令之一。

3. **查找阶段**：当用户输入 `/rewind` 或 `/checkpoint` 时，`findCommand` 函数通过 `name` 或 `aliases` 匹配到此命令定义。

4. **执行阶段**：由于 `type: 'local'`，调用 `load()` 获取模块并执行 `call` 函数。

## 关键代码路径与文件引用

### 依赖关系

```
index.ts
├── 导入: ../../commands.js (Command 类型)
├── 指向: ./rewind.js (实际实现)
└── 被导入于: ../../commands.ts
```

### 调用链

```
用户输入 /rewind
    ↓
REPL.tsx 命令解析
    ↓
commands.ts findCommand()
    ↓
匹配到 rewind 命令定义
    ↓
执行 load() → import('./rewind.js')
    ↓
调用 rewind.ts 中的 call() 函数
    ↓
context.openMessageSelector()
    ↓
REPL.tsx 中 setIsMessageSelectorVisible(true)
    ↓
渲染 MessageSelector 组件
```

## 依赖与外部交互

### 直接依赖

| 依赖 | 路径 | 用途 |
|------|------|------|
| `Command` 类型 | `../../commands.js` | 类型定义和接口约束 |

### 间接依赖（通过命令系统）

| 组件 | 路径 | 交互方式 |
|------|------|----------|
| `REPL.tsx` | `src/screens/REPL.tsx` | 提供 `openMessageSelector` 回调 |
| `MessageSelector` | `src/components/MessageSelector.tsx` | 实际 UI 组件 |
| `fileHistory` | `src/utils/fileHistory.ts` | 代码恢复功能 |

### 被依赖

- `src/commands.ts`：导入并注册此命令
- 命令帮助系统：读取 `description` 显示帮助信息
- 类型提示系统：使用 `name`、`aliases`、`argumentHint` 提供自动补全

## 风险、边界与改进建议

### 当前风险

1. **功能依赖 TUI**：`supportsNonInteractive: false` 意味着在 CI/CD 或脚本环境中无法使用，但缺乏清晰的错误提示。

2. **空参数提示**：`argumentHint: ''` 表示不需要参数，但用户可能期望能直接指定回退到某条消息 ID。

3. **别名认知度**：`checkpoint` 别名可能与某些版本控制概念混淆。

### 边界情况

| 场景 | 行为 |
|------|------|
| 无消息历史 | MessageSelector 显示 "Nothing to rewind to yet" |
| 文件历史禁用 | 仅支持对话恢复，不显示代码恢复选项 |
| 会话恢复后 | 文件历史快照从日志重建，支持跨会话回退 |

### 改进建议

1. **参数支持**：考虑支持 `/rewind <message-id>` 直接跳转到指定消息，跳过选择器。

2. **快捷键绑定**：建议添加默认键盘快捷键（如 `Ctrl+R`）快速打开回退界面。

3. **非交互式支持**：对于非交互式会话，可考虑输出可执行的恢复脚本而非直接失败。

4. **文档增强**：在描述中添加使用示例，如 "Restore to a previous message (e.g., /rewind)"

5. **可见性控制**：考虑添加 `isEnabled` 检查，在文件历史和消息选择器都不可用时禁用此命令。

---

**文件大小**: 337 bytes  
**最后更新**: 基于当前代码库状态分析
