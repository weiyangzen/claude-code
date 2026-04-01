# 研究文档: src/commands/tasks/index.ts

## 场景与职责

`src/commands/tasks/index.ts` 是 `/tasks` (或 `/bashes`) 命令的入口定义文件。该命令属于 **local-jsx** 类型命令，用于在 TUI (Terminal User Interface) 中展示和管理后台任务。

### 使用场景
- 用户输入 `/tasks` 或 `/bashes` 时触发
- 显示当前所有后台运行的任务列表（包括 shell 命令、本地 agent、远程 agent、队友任务、工作流等）
- 提供任务查看、终止、前台化等管理操作

### 核心职责
1. **命令注册**: 定义命令元数据（名称、别名、描述、类型）
2. **懒加载**: 通过 `load()` 函数延迟加载实际实现，减少启动开销
3. **类型约束**: 使用 `satisfies Command` 确保类型安全

---

## 功能点目的

| 功能点 | 目的 |
|--------|------|
| `type: 'local-jsx'` | 标识为本地 JSX 命令，渲染 Ink 组件而非文本输出 |
| `name: 'tasks'` | 主命令名，用户通过 `/tasks` 调用 |
| `aliases: ['bashes']` | 别名 `/bashes`，历史兼容或更直观的 shell 任务管理 |
| `description` | 在帮助系统和命令补全中显示 |
| `load()` | 动态导入实际实现模块，实现代码分割 |

---

## 具体技术实现

### 关键数据结构

```typescript
// 命令定义结构 (来自 src/commands.ts)
interface Command {
  type: 'local-jsx'
  name: string
  aliases?: string[]
  description: string
  load: () => Promise<LocalJSXCommandModule>
}

// JSX 命令模块接口 (来自 src/types/command.ts)
interface LocalJSXCommandModule {
  call: LocalJSXCommandCall
}

type LocalJSXCommandCall = (
  onDone: LocalJSXCommandOnDone,
  context: ToolUseContext & LocalJSXCommandContext,
  args: string,
) => Promise<React.ReactNode>
```

### 加载流程

1. **注册阶段**: 在 `src/commands.ts` 中被导入并加入 `COMMANDS` 数组
2. **调用阶段**: 用户输入 `/tasks` 后，命令系统调用 `tasks.load()`
3. **执行阶段**: 加载 `tasks.tsx`，执行其中的 `call()` 函数返回 React 节点

### 代码实现

```typescript
import type { Command } from '../../commands.js'

const tasks = {
  type: 'local-jsx',
  name: 'tasks',
  aliases: ['bashes'],
  description: 'List and manage background tasks',
  load: () => import('./tasks.js'),
} satisfies Command

export default tasks
```

---

## 关键代码路径与文件引用

### 直接依赖

| 路径 | 用途 |
|------|------|
| `../../commands.js` | 导入 `Command` 类型 |
| `./tasks.js` | 懒加载实际实现（编译后的 `.tsx`） |

### 调用链

```
用户输入 /tasks
    ↓
src/commands.ts 中的命令解析
    ↓
匹配到 tasks 命令
    ↓
调用 tasks.load() → 动态导入 ./tasks.js
    ↓
执行 tasks.tsx 中的 call() 函数
    ↓
渲染 BackgroundTasksDialog 组件
```

### 相关文件

- `src/commands/tasks/tasks.tsx` - 实际实现，渲染 BackgroundTasksDialog
- `src/components/tasks/BackgroundTasksDialog.tsx` - 核心 UI 组件
- `src/commands.ts` - 命令注册中心
- `src/types/command.ts` - 命令类型定义

---

## 依赖与外部交互

### 类型依赖

```typescript
// 来自 ../../commands.js
import type { Command } from '../../commands.js'
```

### 运行时依赖

| 模块 | 说明 |
|------|------|
| `./tasks.js` | 编译后的实现文件，包含 React 组件逻辑 |
| `BackgroundTasksDialog` | 实际渲染任务列表的 Ink 组件 |

### 命令系统集成

该命令通过以下方式集成到系统中：

1. **导入**: 在 `src/commands.ts` 第 45 行导入
   ```typescript
   import tasks from './commands/tasks/index.js'
   ```

2. **注册**: 在 `COMMANDS` 数组第 340 行加入
   ```typescript
   tasks,
   ```

3. **类型**: 作为 `local-jsx` 命令，被 `BRIDGE_SAFE_COMMANDS` 排除（不允许从移动端执行）

---

## 风险、边界与改进建议

### 潜在风险

| 风险 | 说明 | 缓解措施 |
|------|------|----------|
| 循环依赖 | 若 tasks.tsx 反向导入 commands.ts 可能导致循环 | 已通过懒加载避免 |
| 类型不匹配 | `satisfies Command` 在编译时检查，但运行时可能因结构变化失效 | 保持类型定义稳定 |
| 构建产物 | 依赖 `./tasks.js`（编译后），源文件修改需重新构建 | 确保构建流程正确 |

### 边界情况

1. **无任务时**: BackgroundTasksDialog 显示 "No tasks currently running"
2. **单任务优化**: 若只有一个任务，直接进入详情视图而非列表
3. **权限控制**: 某些任务类型（如 workflow）可能因 feature flag 不可用

### 改进建议

1. **类型导入优化**: 当前从 `../../commands.js` 导入类型，可考虑从 `../../types/command.js` 直接导入更精确的类型

2. **元数据扩展**: 可考虑添加 `isHidden: false` 明确可见性，或添加 `availability` 限制特定用户群体

3. **文档注释**: 添加 JSDoc 说明别名历史原因（`/bashes` 与 `/tasks` 的关系）

4. **Feature Flag**: 若未来需要按用户类型控制，可添加：
   ```typescript
   isEnabled: () => feature('BACKGROUND_TASKS'),
   ```

### 测试建议

- 验证命令注册：确保 `getCommands()` 返回的数组包含 tasks
- 验证懒加载：`load()` 返回的模块包含 `call` 函数
- 验证类型：`satisfies Command` 编译通过
