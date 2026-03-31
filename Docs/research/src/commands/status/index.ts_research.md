# 研究文档: src/commands/status/index.ts

## 场景与职责

本文件是 `/status` 命令的入口定义文件，负责声明和导出 status 命令的元数据配置。它是 Claude Code CLI 命令系统中「命令注册」环节的关键组成部分，采用「定义与实现分离」的架构模式：

- **定义层** (`index.ts`): 声明命令类型、名称、描述、加载方式等元数据
- **实现层** (`status.tsx`): 包含实际的 React/Ink 组件渲染逻辑

该命令属于 `local-jsx` 类型命令，意味着它会在终端中渲染交互式 JSX UI（基于 Ink 框架），而非简单的文本输出或 prompt 扩展。

## 功能点目的

### 1. 命令元数据声明
```typescript
const status = {
  type: 'local-jsx',
  name: 'status',
  description: 'Show Claude Code status including version, model, account, API connectivity, and tool statuses',
  immediate: true,
  load: () => import('./status.js'),
} satisfies Command
```

| 属性 | 值 | 说明 |
|------|-----|------|
| `type` | `'local-jsx'` | 命令类型，表示渲染 React/Ink UI |
| `name` | `'status'` | 命令名称，用户输入 `/status` 触发 |
| `description` | 长描述文本 | 在帮助文档和自动补全中显示 |
| `immediate` | `true` | 立即执行，不等待停止点（绕过队列） |
| `load` | 懒加载函数 | 动态导入实际实现，优化启动性能 |

### 2. 懒加载优化
通过 `load: () => import('./status.js')` 实现代码分割，确保：
- 启动时不加载 `status.tsx` 及其依赖（React、Ink、Settings 组件等）
- 仅在用户首次执行 `/status` 时才加载并执行

## 具体技术实现

### 关键数据结构

```typescript
// 来自 src/types/command.ts
interface LocalJSXCommand {
  type: 'local-jsx'
  load: () => Promise<LocalJSXCommandModule>
}

interface LocalJSXCommandModule {
  call: LocalJSXCommandCall
}

type LocalJSXCommandCall = (
  onDone: LocalJSXCommandOnDone,
  context: ToolUseContext & LocalJSXCommandContext,
  args: string,
) => Promise<React.ReactNode>
```

### 命令加载流程

1. **注册阶段** (`src/commands.ts` 第44行):
   ```typescript
   import status from './commands/status/index.js'
   ```
   status 命令被导入并加入 `COMMANDS` 数组

2. **查询阶段** (`getCommands()`):
   命令被过滤、排序后返回给 REPL 系统

3. **执行阶段** (用户输入 `/status`):
   - 命令系统识别到 `type: 'local-jsx'`
   - 调用 `status.load()` 动态导入 `./status.js`
   - 获取模块的 `call` 函数并传入 `onDone` 回调和 `context`
   - Ink 渲染系统接管，显示 Settings 组件（默认选中 Status Tab）

## 关键代码路径与文件引用

### 依赖关系图
```
src/commands/status/index.ts
    │ exports: status (Command)
    │
    ├─import type { Command } ← src/commands.ts
    │
    └─load() → import('./status.js')
                  │
                  └─src/commands/status/status.tsx
                        │
                        ├─import { Settings } ← src/components/Settings/Settings.tsx
                        │   │
                        │   └─import { Status } ← src/components/Settings/Status.tsx
                        │         │
                        │         ├─buildPrimarySection() → 版本、会话名、会话ID、工作目录、账户信息
                        │         ├─buildSecondarySection() → 模型、IDE、MCP、沙盒、设置源
                        │         └─Diagnostics ← 系统诊断信息
                        │
                        └─import type { LocalJSXCommandContext } ← src/types/command.ts
```

### 核心调用链
1. 用户输入 `/status`
2. `src/commands.ts` 中的命令解析器匹配到 `status` 命令
3. 识别 `type: 'local-jsx'`，进入 JSX 命令处理分支
4. 调用 `status.load()` → 加载 `status.tsx`
5. 执行 `call(onDone, context)` 函数
6. 返回 `<Settings onClose={onDone} context={context} defaultTab="Status" />`
7. Ink 渲染器将 React 组件渲染为终端 UI

## 依赖与外部交互

### 直接依赖
| 模块 | 路径 | 用途 |
|------|------|------|
| Command type | `../../commands.js` | 类型约束与接口定义 |
| status.tsx | `./status.js` (编译后) | 实际命令实现 |

### 间接依赖（通过 status.tsx）
| 模块 | 路径 | 用途 |
|------|------|------|
| Settings 组件 | `../../components/Settings/Settings.js` | 设置面板容器 |
| Status 组件 | `../../components/Settings/Status.js` | 状态信息展示 |
| LocalJSXCommandContext | `../../types/command.js` | 命令上下文类型 |

### 与命令系统的集成
- **注册位置**: `src/commands.ts` 第302行 `status,`
- **命令查找**: `findCommand()` / `getCommand()` 函数支持通过 name 或 aliases 查找
- **可用性检查**: 经过 `meetsAvailabilityRequirement()` 和 `isCommandEnabled()` 过滤

## 风险、边界与改进建议

### 潜在风险

1. **编译后路径硬编码**
   - 代码中引用 `./status.js`（而非 `.tsx`），依赖构建系统将 TSX 编译为 JS
   - 如果构建配置变更，可能导致运行时模块找不到错误

2. **immediate: true 的副作用**
   - 命令立即执行，不经过标准队列，可能在某些并发场景下产生竞态条件
   - 但鉴于这是只读的状态展示命令，风险较低

3. **循环依赖风险**
   - `index.ts` 从 `../../commands.js` 导入类型
   - `commands.ts` 又导入 `status` 命令
   - 当前是类型导入（`import type`），不会导致运行时循环依赖

### 边界情况

1. **懒加载失败**
   - 如果 `status.tsx` 或其依赖加载失败，会抛出异常
   - 上层调用者（命令执行器）需要处理这种错误

2. **多次执行**
   - 每次执行 `/status` 都会重新调用 `load()`，但由于模块缓存，实际代码只执行一次
   - 组件状态是全新的（符合预期）

### 改进建议

1. **添加错误边界**
   ```typescript
   load: () => import('./status.js').catch(err => {
     logError(err)
     return { call: () => <Text color="error">Failed to load status</Text> }
   })
   ```

2. **考虑添加 aliases**
   - 如 `st`、`info` 等短别名，提升用户体验
   - 需在 `CommandBase` 中添加 `aliases: ['st']`

3. **类型安全强化**
   - 当前使用 `satisfies Command`，可考虑使用 `as const satisfies Command` 获得更精确的类型推断

4. **文档同步**
   - description 中提到的 "tool statuses" 实际展示在 Settings 组件的 Status Tab 中
   - 确保描述与实际功能保持一致
