# prompt.ts 研究文档

## 场景与职责

prompt.ts 是 EnterWorktreeTool 的提示词定义模块，负责向模型（LLM）提供工具使用指南。这些提示词帮助模型理解何时以及如何使用 EnterWorktreeTool。

**核心场景：**
1. 模型需要判断是否应该调用 EnterWorktreeTool
2. 模型需要了解工具的使用条件和限制
3. 模型需要理解工具的行为和输出

**职责边界：**
- 仅包含提示词文本，不包含逻辑
- 作为模型决策的参考依据
- 与工具实现保持同步

## 功能点目的

### 1. 使用时机指导
- **When to Use**：明确列出应该使用工具的场景
  - 用户显式提及 "worktree"
  - 示例："start a worktree", "work in a worktree", "create a worktree", "use a worktree"

- **When NOT to Use**：明确列出不应该使用工具的场景
  - 用户要求创建/切换分支 → 使用 git 命令
  - 用户要求修复 bug 或开发功能 → 使用常规 git 工作流
  - **关键限制**：除非用户明确提及 "worktree"，否则绝不使用此工具

### 2. 前置条件说明
- 必须在 Git 仓库中，或配置了 WorktreeCreate/WorktreeRemove hooks
- 不能已在 worktree 中

### 3. 行为描述
- 在 Git 仓库中：在 `.claude/worktrees/` 内创建新的 git worktree，基于 HEAD 创建新分支
- 在 Git 仓库外：委托给 WorktreeCreate/WorktreeRemove hooks 进行 VCS 无关的隔离
- 切换会话工作目录到新的 worktree
- 使用 ExitWorktree 退出 worktree

### 4. 参数说明
- `name`（可选）：worktree 名称，未提供时生成随机名称

## 具体技术实现

### 代码实现

```typescript
export function getEnterWorktreeToolPrompt(): string {
  return `Use this tool ONLY when the user explicitly asks to work in a worktree. This tool creates an isolated git worktree and switches the current session into it.

## When to Use

- The user explicitly says "worktree" (e.g., "start a worktree", "work in a worktree", "create a worktree", "use a worktree")

## When NOT to Use

- The user asks to create a branch, switch branches, or work on a different branch — use git commands instead
- The user asks to fix a bug or work on a feature — use normal git workflow unless they specifically mention worktrees
- Never use this tool unless the user explicitly mentions "worktree"

## Requirements

- Must be in a git repository, OR have WorktreeCreate/WorktreeRemove hooks configured in settings.json
- Must not already be in a worktree

## Behavior

- In a git repository: creates a new git worktree inside \`.claude/worktrees/\` with a new branch based on HEAD
- Outside a git repository: delegates to WorktreeCreate/WorktreeRemove hooks for VCS-agnostic isolation
- Switches the session's working directory to the new worktree
- Use ExitWorktree to leave the worktree mid-session (keep or remove). On session exit, if still in the worktree, the user will be prompted to keep or remove it

## Parameters

- \`name\` (optional): A name for the worktree. If not provided, a random name is generated.
`
}
```

### 设计特点

| 特点 | 说明 |
|-----|------|
| 结构清晰 | 使用 Markdown 标题组织内容 |
| 正反对比 | 明确区分 "When to Use" 和 "When NOT to Use" |
| 强调限制 | 使用 "ONLY"、"Never" 等强限制词 |
| 示例具体 | 提供具体的用户请求示例 |
| 参数转义 | 使用 `\`` 转义反引号，确保 Markdown 正确渲染 |

## 关键代码路径与文件引用

### 当前文件
- `/src/tools/EnterWorktreeTool/prompt.ts` - 提示词定义

### 使用位置

| 文件路径 | 使用方式 |
|---------|---------|
| `/src/tools/EnterWorktreeTool/EnterWorktreeTool.ts` | `import { getEnterWorktreeToolPrompt } from './prompt.js'` |

### 使用代码

```typescript
// EnterWorktreeTool.ts
import { getEnterWorktreeToolPrompt } from './prompt.js'

export const EnterWorktreeTool: Tool<InputSchema, Output> = buildTool({
  // ...
  async prompt() {
    return getEnterWorktreeToolPrompt()
  },
  // ...
})
```

### 调用链

```
模型决策
  ↓
系统提示词包含工具描述
  ↓
prompt() 方法被调用
  ↓
getEnterWorktreeToolPrompt() 返回提示词文本
  ↓
模型根据提示词决定是否调用工具
```

## 依赖与外部交互

### 依赖关系

```
prompt.ts
└── (无依赖，纯文本函数)
```

### 与 ExitWorktreeTool 的关系

ExitWorktreeTool 也有对应的提示词定义：

```typescript
// ExitWorktreeTool/prompt.ts
export function getExitWorktreeToolPrompt(): string {
  return `Use this tool when the user wants to exit a worktree session...`
}
```

两个提示词形成互补关系：
- EnterWorktreeTool：如何进入 worktree
- ExitWorktreeTool：如何退出 worktree

## 风险、边界与改进建议

### 已知风险

1. **过度限制**
   - 当前提示词非常严格："Never use this tool unless the user explicitly mentions 'worktree'"
   - 可能导致模型在应该使用 worktree 的场景下不使用
   - 例如：用户说 "create an isolated environment for testing" 可能适合 worktree，但模型不会调用

2. **与实现不同步**
   - 提示词描述的行为必须与 `worktree.ts` 中的实现保持一致
   - 例如：`.claude/worktrees/` 路径、分支命名规则等

3. **模型理解偏差**
   - "worktree" 是 Git 专业术语，普通用户可能不熟悉
   - 用户可能使用其他表达方式（如 "separate workspace", "parallel checkout"）

### 边界情况

1. **多语言支持**
   - 当前提示词仅支持英文
   - 非英语用户可能无法理解 "worktree" 术语

2. **大小写敏感**
   - 提示词未明确说明大小写敏感性
   - "Worktree" vs "worktree" vs "WORKTREE"

3. **复合请求**
   - 用户可能在一个请求中提及多个操作
   - 例如："create a worktree and fix the bug in it"

### 改进建议

1. **扩展触发条件**
   ```typescript
   // 建议：添加更多同义表达
   ## When to Use
   
   - The user explicitly says "worktree" (e.g., "start a worktree", "work in a worktree", "create a worktree", "use a worktree")
   - The user asks for an "isolated workspace" or "separate working directory" for parallel development
   - The user wants to work on multiple branches simultaneously without stashing
   ```

2. **添加示例场景**
   ```typescript
   // 建议：添加具体使用场景示例
   ## Example Scenarios
   
   - User: "I want to test a refactoring idea without affecting my current work"
   - User: "Can I work on the bug fix while keeping my feature branch as-is?"
   - User: "Create a worktree for the hotfix so I don't have to commit my WIP"
   ```

3. **澄清与分支的区别**
   ```typescript
   // 建议：更清晰地解释 worktree 与分支的区别
   ## Worktree vs Branch
   
   - A branch is just a pointer to a commit; you can only be on one branch at a time in a single working directory
   - A worktree is a complete working directory with its own branch checked out
   - Use branches for simple context switching; use worktrees for parallel work without stashing
   ```

4. **添加错误处理提示**
   ```typescript
   // 建议：告知模型如何处理错误情况
   ## Error Handling
   
   - If the tool returns "Already in a worktree session", inform the user they must exit the current worktree first
   - If not in a git repository, suggest using git init or configuring WorktreeCreate hooks
   ```

5. **国际化支持**
   ```typescript
   // 建议：支持多语言提示词
   export function getEnterWorktreeToolPrompt(locale: string = 'en'): string {
     const prompts = {
       en: `...`,
       zh: `...`,
       // ...
     }
     return prompts[locale] ?? prompts.en
   }
   ```

6. **动态路径提示**
   ```typescript
   // 建议：如果配置支持，动态显示实际路径
   import { getInitialSettings } from '../../utils/settings/settings.js'
   
   export function getEnterWorktreeToolPrompt(): string {
     const settings = getInitialSettings()
     const worktreeDir = settings.worktree?.directory ?? '.claude/worktrees/'
     return `...creates a new git worktree inside \`${worktreeDir}\`...`
   }
   ```

### 与系统提示词的集成

工具提示词通过 `buildTool.prompt()` 方法集成到系统提示词中：

```typescript
// 系统提示词结构示例
const systemPrompt = `
You have access to the following tools:

## EnterWorktree
${await EnterWorktreeTool.prompt()}

## ExitWorktree
${await ExitWorktreeTool.prompt()}

...其他工具
`
```

确保工具提示词与整体系统提示词风格一致，避免冲突或重复。
