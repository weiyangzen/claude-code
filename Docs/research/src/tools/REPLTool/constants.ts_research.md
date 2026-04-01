# REPLTool/constants.ts 深度研究文档

## 场景与职责

`constants.ts` 是 REPL（Read-Eval-Print Loop）工具模块的核心常量定义文件，负责：

1. **定义 REPL 工具名称常量** - `REPL_TOOL_NAME = 'REPL'`
2. **实现 REPL 模式启用检测逻辑** - `isReplModeEnabled()` 函数
3. **定义 REPL 独占工具集合** - `REPL_ONLY_TOOLS` Set

该文件是 REPL 模式的"开关中心"，决定了何时启用 REPL 模式以及哪些基础工具在 REPL 模式下对模型直接隐藏。

## 功能点目的

### 1. REPL 工具名称常量
```typescript
export const REPL_TOOL_NAME = 'REPL'
```
- 作为 REPL 工具的唯一标识符
- 被 `src/tools.ts`、`src/utils/collapseReadSearch.ts` 等多个模块引用

### 2. REPL 模式启用检测 (`isReplModeEnabled`)

**设计目标**：
- 为交互式 CLI 默认启用 REPL 模式（蚂蚁内部环境）
- 允许用户通过环境变量显式控制
- SDK 入口点默认禁用（避免破坏 SDK 用户的脚本化工具调用）

**检测优先级**（从高到低）：
1. `CLAUDE_CODE_REPL=0` → 强制禁用（无论其他设置）
2. `CLAUDE_REPL_MODE=1` → 强制启用（向后兼容旧变量）
3. `USER_TYPE === 'ant' && CLAUDE_CODE_ENTRYPOINT === 'cli'` → 蚂蚁内部 CLI 默认启用

**环境变量说明**：
- `CLAUDE_CODE_REPL`：主控开关，设为 `0`/`false`/`no`/`off` 可禁用
- `CLAUDE_REPL_MODE`：遗留变量，设为 `1`/`true`/`yes`/`on` 强制启用
- `USER_TYPE`：构建时定义，`'ant'` 表示蚂蚁内部构建
- `CLAUDE_CODE_ENTRYPOINT`：入口点标识，`'cli'` 表示交互式 CLI

### 3. REPL 独占工具集合 (`REPL_ONLY_TOOLS`)

**包含的工具**（8个基础工具）：
| 工具名 | 来源常量 | 用途 |
|--------|----------|------|
| Read | `FILE_READ_TOOL_NAME` | 文件读取 |
| Write | `FILE_WRITE_TOOL_NAME` | 文件写入 |
| Edit | `FILE_EDIT_TOOL_NAME` | 文件编辑 |
| Glob | `GLOB_TOOL_NAME` | 文件模式匹配 |
| Grep | `GREP_TOOL_NAME` | 文本搜索 |
| Bash | `BASH_TOOL_NAME` | 命令执行 |
| NotebookEdit | `NOTEBOOK_EDIT_TOOL_NAME` | Notebook 编辑 |
| Agent | `AGENT_TOOL_NAME` | 子代理调用 |

**设计意图**：
- 当 REPL 模式启用时，这些工具从模型的直接可见工具列表中隐藏
- 强制模型通过 REPL 工具执行批量化操作（在 VM 中调用这些基础工具）
- 基础工具在 REPL VM 上下文中仍然可用

## 具体技术实现

### 依赖的工具名称常量

```typescript
import { AGENT_TOOL_NAME } from '../AgentTool/constants.js'
import { BASH_TOOL_NAME } from '../BashTool/toolName.js'
import { FILE_EDIT_TOOL_NAME } from '../FileEditTool/constants.js'
import { FILE_READ_TOOL_NAME } from '../FileReadTool/prompt.js'
import { FILE_WRITE_TOOL_NAME } from '../FileWriteTool/prompt.js'
import { GLOB_TOOL_NAME } from '../GlobTool/prompt.js'
import { GREP_TOOL_NAME } from '../GrepTool/prompt.js'
import { NOTEBOOK_EDIT_TOOL_NAME } from '../NotebookEditTool/constants.js'
```

### 环境变量检测工具函数

```typescript
// src/utils/envUtils.ts
export function isEnvTruthy(envVar: string | boolean | undefined): boolean
export function isEnvDefinedFalsy(envVar: string | boolean | undefined): boolean
```

- `isEnvTruthy`：识别 `'1'`, `'true'`, `'yes'`, `'on'` 为真
- `isEnvDefinedFalsy`：识别 `'0'`, `'false'`, `'no'`, `'off'` 为假，且变量必须已定义

### REPL 模式检测流程

```
isReplModeEnabled()
├── CLAUDE_CODE_REPL 已定义且为假值? → return false
├── CLAUDE_REPL_MODE 为真值? → return true
└── return (USER_TYPE === 'ant' && CLAUDE_CODE_ENTRYPOINT === 'cli')
```

## 关键代码路径与文件引用

### 被调用方（消费者）

| 文件路径 | 使用方式 | 用途 |
|----------|----------|------|
| `src/tools.ts` | 导入 `REPL_TOOL_NAME`, `REPL_ONLY_TOOLS`, `isReplModeEnabled` | 工具列表过滤、REPL 工具注册 |
| `src/utils/collapseReadSearch.ts` | 导入 `REPL_TOOL_NAME` | 消息折叠逻辑中的 REPL 检测 |
| `src/memdir/memdir.ts` | 导入 `isReplModeEnabled` | 内存搜索提示生成 |
| `src/constants/prompts.ts` | 导入 `isReplModeEnabled` | 系统提示词生成（REPL 模式下的工具使用指导） |
| `src/components/messages/CollapsedReadSearchContent.tsx` | 通过 `getReplPrimitiveTools` 间接使用 | 渲染折叠消息 |

### 调用方（依赖）

| 文件路径 | 导入内容 | 说明 |
|----------|----------|------|
| `src/utils/envUtils.ts` | `isEnvTruthy`, `isEnvDefinedFalsy` | 环境变量检测工具 |
| `src/tools/AgentTool/constants.ts` | `AGENT_TOOL_NAME` | Agent 工具名称 |
| `src/tools/BashTool/toolName.ts` | `BASH_TOOL_NAME` | Bash 工具名称 |
| `src/tools/FileEditTool/constants.ts` | `FILE_EDIT_TOOL_NAME` | 文件编辑工具名称 |
| `src/tools/FileReadTool/prompt.ts` | `FILE_READ_TOOL_NAME` | 文件读取工具名称 |
| `src/tools/FileWriteTool/prompt.ts` | `FILE_WRITE_TOOL_NAME` | 文件写入工具名称 |
| `src/tools/GlobTool/prompt.ts` | `GLOB_TOOL_NAME` | Glob 工具名称 |
| `src/tools/GrepTool/prompt.ts` | `GREP_TOOL_NAME` | Grep 工具名称 |
| `src/tools/NotebookEditTool/constants.ts` | `NOTEBOOK_EDIT_TOOL_NAME` | Notebook 编辑工具名称 |

## 依赖与外部交互

### 构建时定义
- `USER_TYPE`：通过构建系统的 `--define` 注入
- `CLAUDE_CODE_ENTRYPOINT`：运行时根据入口点设置

### 运行时环境变量
- `CLAUDE_CODE_REPL`：用户可配置的主控开关
- `CLAUDE_REPL_MODE`：向后兼容的遗留开关

### 与 tools.ts 的协作

在 `src/tools.ts` 中：

1. **REPL 工具条件注册**（第 232 行）：
```typescript
...(process.env.USER_TYPE === 'ant' && REPLTool ? [REPLTool] : [])
```

2. **工具列表过滤**（第 314-322 行）：
```typescript
if (isReplModeEnabled()) {
  const replEnabled = allowedTools.some(tool => toolMatchesName(tool, REPL_TOOL_NAME))
  if (replEnabled) {
    allowedTools = allowedTools.filter(tool => !REPL_ONLY_TOOLS.has(tool.name))
  }
}
```

3. **Simple 模式处理**（第 273-286 行）：
```typescript
if (isReplModeEnabled() && REPLTool) {
  const replSimple: Tool[] = [REPLTool]
  // ... 协调器模式额外添加 TaskStopTool 和 SendMessageTool
  return filterToolsByDenyRules(replSimple, permissionContext)
}
```

## 风险、边界与改进建议

### 潜在风险

1. **环境变量优先级混淆**
   - `CLAUDE_CODE_REPL=0` 优先级最高，但用户可能误以为是 `CLAUDE_REPL_MODE=0` 也能禁用
   - 建议：统一环境变量命名规范， deprecate 旧变量

2. **SDK 用户意外启用**
   - 如果 SDK 入口点错误地设置了 `CLAUDE_CODE_ENTRYPOINT=cli`，会导致 REPL 模式意外启用
   - 影响：SDK 用户的脚本化工具调用会被隐藏，破坏预期行为

3. **工具集合硬编码**
   - `REPL_ONLY_TOOLS` 是编译时确定的 Set，新增基础工具时需要手动同步
   - 风险：忘记更新会导致新工具在 REPL 模式下仍对模型可见

### 边界情况

1. **REPL 工具未注册但模式启用**
   - 如果 `REPLTool` 因 `USER_TYPE !== 'ant'` 未注册，但 `isReplModeEnabled()` 返回 true
   - `tools.ts` 第 315-316 行会检查 `allowedTools.some(toolMatchesName(tool, REPL_TOOL_NAME))`，避免误过滤

2. **协调器模式与 REPL 模式共存**
   - 在 `--bare` + REPL 模式下，协调器需要额外的 `TaskStopTool` 和 `SendMessageTool`
   - 代码已处理（`tools.ts` 第 279-284 行）

3. **嵌入式搜索工具影响**
   - 蚂蚁内部构建使用嵌入式 bfs/ugrep，此时 `GlobTool` 和 `GrepTool` 不会出现在基础工具列表
   - `REPL_ONLY_TOOLS` 仍包含它们，但过滤时无实际影响

### 改进建议

1. **类型安全增强**
   ```typescript
   // 建议使用 const assertion 和类型导出
   export const REPL_ONLY_TOOLS = [...] as const
   export type REPLOnlyTool = typeof REPL_ONLY_TOOLS[number]
   ```

2. **环境变量文档化**
   - 在代码中添加更详细的 JSDoc 说明环境变量优先级
   - 考虑添加运行时警告当检测到冲突的环境变量设置

3. **工具集合自动化验证**
   - 添加测试用例验证 `REPL_ONLY_TOOLS` 与 `getReplPrimitiveTools()` 返回的工具列表一致
   - 在 CI 中检查新增基础工具时是否同步更新了 `REPL_ONLY_TOOLS`

4. **配置合并优化**
   - 考虑将 REPL 模式配置统一到一个配置对象中，而非分散的环境变量检查
   - 示例：`{ enabled: boolean, source: 'env' | 'default' | 'legacy' }`
