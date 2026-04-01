# constants.ts 研究文档

## 场景与职责

`constants.ts` 是 `FileEditTool` 模块的常量定义文件，设计意图是“将常量独立出来以避免循环依赖”（文件顶部注释明确说明）。该文件虽然体积极小（约 500 字节），但在整个 Claude Code 仓库中被广泛引用，是权限系统、提示词系统、工具执行系统等多个子模块共享的“协议锚点”。

核心职责：
- 定义 `FileEditTool` 的对外名称常量 `FILE_EDIT_TOOL_NAME`。
- 定义 `.claude/` 文件夹的权限模式常量，用于会话级权限授予。
- 定义文件意外修改时的标准化错误文案。

## 功能点目的

| 常量 | 目的 |
|------|------|
| `FILE_EDIT_TOOL_NAME` | 统一工具名称 `'Edit'`，作为工具注册、权限规则匹配、提示词引用、遥测标识的单一事实来源。 |
| `CLAUDE_FOLDER_PERMISSION_PATTERN` | 项目级 `.claude/` 目录的权限通配模式（`'/.claude/**'`），用于权限对话框和 SDK 建议中的会话级允许规则。 |
| `GLOBAL_CLAUDE_FOLDER_PERMISSION_PATTERN` | 全局 `~/.claude/` 目录的权限通配模式（`'~/.claude/**'`），同上，但作用于用户主目录。 |
| `FILE_UNEXPECTEDLY_MODIFIED_ERROR` | 原子写操作中发现文件在读取后被外部修改时抛出的错误消息，保证 `FileEditTool.ts` 与 `FileWriteTool.ts` 文案一致。 |

## 具体技术实现

文件内容极为精简，共 4 个导出常量：

```ts
// In its own file to avoid circular dependencies
export const FILE_EDIT_TOOL_NAME = 'Edit'

// Permission pattern for granting session-level access to the project's .claude/ folder
export const CLAUDE_FOLDER_PERMISSION_PATTERN = '/.claude/**'

// Permission pattern for granting session-level access to the global ~/.claude/ folder
export const GLOBAL_CLAUDE_FOLDER_PERMISSION_PATTERN = '~/.claude/**'

export const FILE_UNEXPECTEDLY_MODIFIED_ERROR =
  'File has been unexpectedly modified. Read it again before attempting to write it.'
```

### 1. `FILE_EDIT_TOOL_NAME = 'Edit'`

这是模型可见的工具名称，也是权限系统中 `edit` 类型规则的关联键。在 `src/utils/permissions/filesystem.ts` 的 `getPatternsByRoot` 函数中，工具类型到工具名的映射逻辑为：

```ts
case 'edit':
  return FILE_EDIT_TOOL_NAME  // 'Edit'
```

这意味着所有针对 `Edit` 工具的权限规则（allow/deny/ask）都通过该字符串索引。若此处改名，权限系统、提示词、工具注册、测试用例均需同步更新。

### 2. 权限模式常量

`CLAUDE_FOLDER_PERMISSION_PATTERN` 和 `GLOBAL_CLAUDE_FOLDER_PERMISSION_PATTERN` 在 `src/utils/permissions/filesystem.ts` 的 `checkWritePermissionForTool` 中被使用：

```ts
const claudeFolderAllowRule = matchingRuleForInput(
  path,
  { ...toolPermissionContext, alwaysAllowRules: { session: ... } },
  'edit',
  'allow',
)
// 后续校验规则内容是否以模式前缀开头
ruleContent.startsWith(CLAUDE_FOLDER_PERMISSION_PATTERN.slice(0, -2)) ||
ruleContent.startsWith(GLOBAL_CLAUDE_FOLDER_PERMISSION_PATTERN.slice(0, -2))
```

`.slice(0, -2)` 去掉末尾的 `**`，从而同时匹配 `'/.claude/**'` 和 `'/.claude/skills/my-skill/**'` 等更细粒度的规则。

### 3. `FILE_UNEXPECTEDLY_MODIFIED_ERROR`

在 `FileEditTool.ts` 的 `call` 方法中，当二次读取文件后发现 `mtimeMs` 变更且 content fallback 失败时，抛出该错误：

```ts
if (!contentUnchanged) {
  throw new Error(FILE_UNEXPECTEDLY_MODIFIED_ERROR)
}
```

`FileWriteTool.ts` 也导入并使用了同一常量，确保两种写工具的错误文案完全一致，便于客户端统一识别和处理。

## 关键代码路径与文件引用

- **定义文件**：`src/tools/FileEditTool/constants.ts`
- **主要引用方**（通过 `grep` 不完全统计，仓库中超过 30 处）：
  - `src/tools.ts` — 工具注册
  - `src/constants/prompts.ts` — 系统提示词拼接
  - `src/constants/tools.ts` — 工具常量聚合
  - `src/services/tools/toolExecution.ts` — 工具执行逻辑
  - `src/services/MagicDocs/magicDocs.ts` — MagicDocs 功能
  - `src/services/SessionMemory/sessionMemory.ts` — 会话记忆
  - `src/services/autoDream/autoDream.ts` — AutoDream
  - `src/services/extractMemories/prompts.ts` / `extractMemories.ts` — 记忆提取
  - `src/services/compact/microCompact.ts` / `apiMicrocompact.ts` — 消息压缩
  - `src/utils/collapseReadSearch.ts` — 读/搜索折叠
  - `src/utils/teamMemoryOps.ts` — 团队记忆操作
  - `src/utils/attribution.ts` — 归因统计
  - `src/utils/queryHelpers.ts` — 查询辅助
  - `src/utils/streamlinedTransform.ts` — 流式转换
  - `src/utils/sessionFileAccessHooks.ts` — 会话文件访问 hook
  - `src/utils/messages.ts` — 消息处理
  - `src/utils/api.ts` — API 工具输入归一化
  - `src/utils/plugins/loadPluginAgents.ts` — 插件代理加载
  - `src/utils/sandbox/sandbox-adapter.ts` — 沙箱适配
  - `src/utils/permissions/filesystem.ts` — 权限系统（核心消费方）
  - `src/tools/BashTool/prompt.ts` — Bash 工具提示词
  - `src/tools/BashTool/BashTool.tsx` — Bash 工具 UI
  - `src/tools/PowerShellTool/prompt.ts` — PowerShell 提示词
  - `src/tools/TodoWriteTool/prompt.ts` — TodoWrite 提示词
  - `src/tools/REPLTool/constants.ts` / `primitiveTools.ts` — REPL 工具
  - `src/tools/AgentTool/built-in/*.ts` — 各类内置 Agent
  - `src/tools/FileWriteTool/FileWriteTool.ts` — 文件写工具
  - `src/hooks/useDiffInIDE.ts` — IDE diff hook
  - `src/hooks/useTurnDiffs.ts` — 轮次 diff
  - `src/components/MessageSelector.tsx` — 消息选择器
  - `src/components/permissions/PermissionRequest.tsx` — 权限请求 UI
  - `src/components/agents/ToolSelector.tsx` — 工具选择器
  - `src/coordinator/coordinatorMode.ts` — 协调器模式

## 依赖与外部交互

该文件本身无任何导入依赖，是纯常量导出模块。所有交互均为“被依赖”关系：

| 消费方 | 消费内容 | 说明 |
|--------|----------|------|
| 权限系统 (`filesystem.ts`) | `FILE_EDIT_TOOL_NAME` + 两个 `PERMISSION_PATTERN` | 工具名映射、`.claude/` 会话权限白名单。 |
| 工具注册 (`tools.ts`) | `FILE_EDIT_TOOL_NAME` | 构建全局工具列表。 |
| 提示词系统 (`prompts.ts`, `BashTool/prompt.ts` 等) | `FILE_EDIT_TOOL_NAME` | 在系统提示中引用 Edit 工具，指导模型使用。 |
| 写工具 (`FileWriteTool.ts`) | `FILE_UNEXPECTEDLY_MODIFIED_ERROR` | 统一外部修改错误文案。 |
| 各类服务/工具/组件 | `FILE_EDIT_TOOL_NAME` | 遥测、记忆提取、消息压缩、Agent 定义等。 |

## 风险、边界与改进建议

### 风险与边界

1. **循环依赖的历史包袱**：文件注释明确指出“In its own file to avoid circular dependencies”。这说明早期架构中 `FileEditTool.ts` 与权限/提示词模块存在循环引用风险，当前通过将常量抽离打破循环。若未来不慎将业务逻辑重新引入 `constants.ts`，可能重新引入循环依赖。
2. **字符串常量的“隐式协议”**：`FILE_EDIT_TOOL_NAME = 'Edit'` 是跨模块的隐式协议键。若重命名（如改为 `'FileEdit'`），需要全仓库同步更新，包括用户侧的持久化权限规则（若用户 settings 中已存储 `'Edit'` 规则，则会产生兼容性问题）。
3. **权限模式的硬编码前缀**：`/.claude/**` 和 `~/.claude/**` 是写死的字符串，未与 `src/utils/envUtils.ts` 中的 `getClaudeConfigHomeDir()` 等动态逻辑关联。若未来 `.claude` 目录命名变更，需要多处手动同步。
4. **错误文案的国际化缺失**：`FILE_UNEXPECTEDLY_MODIFIED_ERROR` 为英文硬编码，当前系统无 i18n 框架，非英语用户可能体验不佳。

### 改进建议

1. **增加类型安全包装**：为 `FILE_EDIT_TOOL_NAME` 定义一个 branded type 或 enum，防止在权限系统中误传其他字符串。例如：
   ```ts
   export type EditToolName = typeof FILE_EDIT_TOOL_NAME
   ```
2. **将 `.claude` 目录名提取为单一常量**：在更底层的配置模块（如 `envUtils.ts`）中定义 `CLAUDE_DIR_NAME = '.claude'`，然后 `CLAUDE_FOLDER_PERMISSION_PATTERN = `/${CLAUDE_DIR_NAME}/**``，避免硬编码分散。
3. **文档化重命名流程**：若未来确实需要重命名工具，应在 `AGENTS.md` 或内部文档中建立 checklist，涵盖工具注册、权限系统、提示词、测试 fixture、用户持久化规则迁移脚本等。
4. **考虑错误文案的常量聚合**：`FILE_UNEXPECTEDLY_MODIFIED_ERROR` 目前仅被两个文件使用，但随着写工具增多（如 `NotebookEditTool`），可能也需要共享。可评估是否提升到 `src/constants/errors.ts` 级别的共享错误文案库。
