# src/commands/export/index.ts 研究文档

## 场景与职责

本文件是 Claude Code `/export` 命令的入口定义文件，采用命令注册模式将导出功能集成到系统的命令体系中。作为 `local-jsx` 类型的命令，它遵循懒加载（lazy-loading）设计模式，只在用户实际调用时才加载实现代码。

**核心职责：**
- 定义 `/export` 命令的元数据（名称、描述、参数提示）
- 指定命令类型为 `local-jsx`（本地 JSX 交互式命令）
- 提供懒加载入口，延迟加载实际的命令实现

## 功能点目的

### 1. 命令注册
将导出功能注册到系统的命令体系中，使其可以通过 `/export` 斜杠命令调用。

### 2. 懒加载优化
通过 `load: () => import('./export.js')` 实现动态导入，避免在启动时加载不必要的代码，减少初始启动时间和内存占用。

### 3. 类型安全
使用 TypeScript 的 `satisfies Command` 确保命令定义符合系统要求的 `Command` 接口规范。

## 具体技术实现

### 代码结构

```typescript
import type { Command } from '../../commands.js'

const exportCommand = {
  type: 'local-jsx',
  name: 'export',
  description: 'Export the current conversation to a file or clipboard',
  argumentHint: '[filename]',
  load: () => import('./export.js'),
} satisfies Command

export default exportCommand
```

### 关键字段说明

| 字段 | 类型 | 说明 |
|------|------|------|
| `type` | `'local-jsx'` | 命令类型，表示这是一个本地执行的 JSX 交互式命令 |
| `name` | `'export'` | 命令名称，用户通过 `/export` 调用 |
| `description` | `string` | 命令描述，显示在帮助和自动补全中 |
| `argumentHint` | `'[filename]'` | 参数提示，方括号表示可选参数 |
| `load` | `() => Promise<LocalJSXCommandModule>` | 懒加载函数，返回命令实现模块 |

### 类型定义

```typescript
// 来自 ../../commands.js
interface Command {
  type: 'local-jsx' | 'local' | 'prompt'
  name: string
  description: string
  argumentHint?: string
  load: () => Promise<CommandModule>
  // ... 其他可选字段
}

// local-jsx 类型特有的模块结构
interface LocalJSXCommandModule {
  call: (
    onDone: LocalJSXCommandOnDone,
    context: ToolUseContext & LocalJSXCommandContext,
    args: string
  ) => Promise<React.ReactNode>
}
```

## 关键代码路径与文件引用

### 导入依赖

```
src/commands/export/index.ts
    └─→ src/commands.ts (Command 类型)
```

### 懒加载目标

```
src/commands/export/index.ts
    load() → import('./export.js')
                  ↓
           src/commands/export/export.tsx
```

### 注册流程

```
src/commands/export/index.ts (export default exportCommand)
    ↓ 导入
src/commands.ts
    ├─→ import exportCommand from './commands/export/index.js'
    ├─→ 添加到 COMMANDS() 数组
    └─→ 通过 getCommands() 暴露给系统
```

## 依赖与外部交互

### 静态依赖

| 模块 | 路径 | 用途 |
|------|------|------|
| Command 类型 | `../../commands.js` | 类型检查和接口约束 |

### 动态依赖

| 模块 | 路径 | 加载时机 | 用途 |
|------|------|----------|------|
| export.js | `./export.js` | 命令被调用时 | 实际命令实现 |

### 与命令系统的交互

1. **注册阶段**
   - `src/commands.ts` 静态导入 `exportCommand`
   - 添加到内置命令列表 `COMMANDS()`

2. **调用阶段**
   - 用户输入 `/export`
   - 命令解析器匹配到 `exportCommand`
   - 调用 `load()` 动态加载 `./export.js`
   - 执行导出的 `call()` 函数

3. **完成阶段**
   - `call()` 返回 `React.ReactNode`（对话框组件）或 `null`
   - 通过 `onDone` 回调通知命令完成

## 风险、边界与改进建议

### 潜在风险

1. **模块路径变更**
   - 如果 `export.tsx` 文件位置变更，需要同步更新 `load()` 中的导入路径
   - 编译后的文件扩展名从 `.tsx` 变为 `.js`，路径需注意

2. **类型不匹配**
   - 如果 `export.tsx` 中的 `call` 函数签名变更，但此处未更新类型定义，可能导致运行时错误
   - `satisfies Command` 提供编译时检查，但不能完全防止实现不匹配

### 边界情况

1. **懒加载失败**
   - 如果 `export.tsx` 文件损坏或缺失，`import()` 会抛出错误
   - 错误处理由调用方（命令调度器）负责

2. **循环依赖**
   - 当前设计避免了循环依赖，因为实现模块是动态加载的
   - 但如果在 `export.tsx` 中导入此文件会导致问题

### 改进建议

1. **添加加载错误处理**
   ```typescript
   load: async () => {
     try {
       return await import('./export.js')
     } catch (error) {
       logError('Failed to load export command:', error)
       throw new Error('Export command is temporarily unavailable')
     }
   }
   ```

2. **预加载优化**
   - 对于常用命令，可考虑在空闲时预加载
   ```typescript
   // 在适当的时机预加载
   if (typeof window !== 'undefined') {
     requestIdleCallback(() => import('./export.js'))
   }
   ```

3. **类型导出**
   - 导出命令特定的类型，便于其他模块使用
   ```typescript
   export type ExportCommandArgs = {
     filename?: string
   }
   ```

4. **元数据扩展**
   - 考虑添加更多元数据，如：
     ```typescript
     {
       category: 'utility',
       aliases: ['save', 'download'],
       examples: ['/export', '/export my-conversation.txt']
     }
     ```

5. **权限控制**
   - 如果未来需要限制导出功能，可添加 `isEnabled` 检查
   ```typescript
   isEnabled: () => !isNonInteractiveSession()
   ```

### 架构一致性

此文件遵循 Claude Code 的命令定义模式，与系统中其他命令保持一致：

```typescript
// 类似结构的命令定义示例
const copyCommand = {
  type: 'local-jsx',
  name: 'copy',
  description: 'Copy the last message to clipboard',
  load: () => import('./copy.js'),
} satisfies Command
```

这种模式确保了：
- 统一的命令注册机制
- 一致的懒加载行为
- 类型安全的命令定义
