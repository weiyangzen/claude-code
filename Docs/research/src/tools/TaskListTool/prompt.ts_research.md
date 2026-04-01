# prompt.ts 研究文档

## 场景与职责

`prompt.ts` 负责生成 TaskListTool 的描述文本和提示词内容。它是工具与 AI 模型交互的桥梁，通过精心设计的提示词指导模型何时以及如何使用 TaskList 工具。

主要场景：
1. **工具发现**：提供工具描述（description）供模型理解工具用途
2. **使用指导**：生成详细的提示词（prompt）指导模型正确使用工具
3. **Agent Swarms 适配**：根据 teammate 功能开关动态调整提示词内容

## 功能点目的

### 1. 工具描述（DESCRIPTION）
提供简洁的工具功能描述，用于工具注册和模型理解：
```typescript
export const DESCRIPTION = 'List all tasks in the task list'
```

### 2. 动态提示词生成（getPrompt）
根据运行环境（特别是 Agent Swarms 是否启用）生成差异化的使用指南：

- **基础使用场景**：列出所有用户使用场景
- **输出字段说明**：解释返回数据的每个字段含义
- **Teammate 工作流**（可选）：当 Agent Swarms 启用时，添加 teammate 专属使用指南

## 具体技术实现

### 核心函数

```typescript
export function getPrompt(): string {
  const teammateUseCase = isAgentSwarmsEnabled()
    ? `- Before assigning tasks to teammates, to see what's available\n`
    : ''

  const idDescription = isAgentSwarmsEnabled()
    ? '- **id**: Task identifier (use with TaskGet, TaskUpdate)'
    : '- **id**: Task identifier (use with TaskGet, TaskUpdate)'

  const teammateWorkflow = isAgentSwarmsEnabled()
    ? `\n## Teammate Workflow\n...`
    : ''

  return `Use this tool to list all tasks in the task list.

## When to Use This Tool

- To see what tasks are available to work on (status: 'pending', no owner, not blocked)
- To check overall progress on the project
- To find tasks that are blocked and need dependencies resolved
${teammateUseCase}- After completing a task, to check for newly unblocked work or claim the next available task
- **Prefer working on tasks in ID order** (lowest ID first) when multiple tasks are available, as earlier tasks often set up context for later ones

## Output

Returns a summary of each task:
${idDescription}
- **subject**: Brief description of the task
- **status**: 'pending', 'in_progress', or 'completed'
- **owner**: Agent ID if assigned, empty if available
- **blockedBy**: List of open task IDs that must be resolved first (tasks with blockedBy cannot be claimed until dependencies resolve)

Use TaskGet with a specific task ID to view full details including description and comments.
${teammateWorkflow}`
}
```

### 动态内容逻辑

| 条件变量 | Agent Swarms 启用 | Agent Swarms 禁用 |
|----------|-------------------|-------------------|
| `teammateUseCase` | 包含分配任务给 teammate 的场景 | 空字符串 |
| `teammateWorkflow` | 完整的 teammate 工作流指南 | 空字符串 |

### Teammate 工作流详情

当 Agent Swarms 启用时，提示词包含以下 teammate 专属指南：

```markdown
## Teammate Workflow

When working as a teammate:
1. After completing your current task, call TaskList to find available work
2. Look for tasks with status 'pending', no owner, and empty blockedBy
3. **Prefer tasks in ID order** (lowest ID first) when multiple tasks are available, as earlier tasks often set up context for later ones
4. Claim an available task using TaskUpdate (set `owner` to your name), or wait for leader assignment
5. If blocked, focus on unblocking tasks or notify the team lead
```

## 依赖与外部交互

### 直接依赖

| 依赖 | 路径 | 用途 |
|------|------|------|
| `isAgentSwarmsEnabled` | `../../utils/agentSwarmsEnabled.js` | 功能开关检查 |

### 依赖函数实现

**agentSwarmsEnabled.ts**（简化）：
```typescript
export function isAgentSwarmsEnabled(): boolean {
  // Ant: always on
  if (process.env.USER_TYPE === 'ant') {
    return true
  }
  // External: require opt-in via env var or --agent-teams flag
  if (!isEnvTruthy(process.env.CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS) &&
      !isAgentTeamsFlagSet()) {
    return false
  }
  // Killswitch check
  if (!getFeatureValue_CACHED_MAY_BE_STALE('tengu_amber_flint', true)) {
    return false
  }
  return true
}
```

### 调用方

| 文件 | 用途 |
|------|------|
| `src/tools/TaskListTool/TaskListTool.ts` | 工具的 `description()` 和 `prompt()` 方法 |

**TaskListTool.ts** 中的使用（行 37-42）：
```typescript
async description() {
  return DESCRIPTION
},
async prompt() {
  return getPrompt()
},
```

## 风险、边界与改进建议

### 风险点

1. **提示词膨胀**：随着功能增加，提示词可能变得冗长，增加 token 消耗
2. **条件复杂性**：`isAgentSwarmsEnabled()` 依赖多个条件（环境变量、CLI 参数、GrowthBook），可能在不同环境表现不一致
3. **国际化缺失**：提示词为硬编码英文，不支持多语言

### 边界情况

1. **功能开关延迟**：`isAgentSwarmsEnabled()` 使用缓存的 GrowthBook 值，可能在功能切换后有短暂延迟
2. **提示词模板注入**：虽然当前无用户输入直接插入，但未来扩展时需注意 XSS/注入风险

### 改进建议

1. **添加使用示例**：
   ```typescript
   const examples = `
   ## Examples
   
   List all tasks:
   <thinking>The user wants to see all tasks. I'll use TaskList.</thinking>
   
   ✅ Result:
   #1 [pending] Fix login bug
   #2 [in_progress] Add tests (alice)
   #3 [completed] Update documentation
   `
   ```

2. **优化提示词结构**：
   - 使用更清晰的层级结构
   - 添加 "Common Mistakes" 部分避免误用

3. **考虑提示词版本控制**：
   ```typescript
   export const PROMPT_VERSION = '1.0.0'
   ```

4. **添加性能提示**（如果任务量大）：
   ```markdown
   ## Performance Note
   
   If the task list is very large, consider using TaskGet for specific tasks
   instead of listing all tasks.
   ```

5. **缓存提示词结果**：
   ```typescript
   let cachedPrompt: string | undefined
   export function getPrompt(): string {
     if (cachedPrompt) return cachedPrompt
     cachedPrompt = buildPrompt()
     return cachedPrompt
   }
   ```

### 相关文件引用

- 提示词实现：`src/tools/TaskListTool/prompt.ts`
- 工具实现：`src/tools/TaskListTool/TaskListTool.ts`
- 功能开关：`src/utils/agentSwarmsEnabled.ts`
- GrowthBook 服务：`src/services/analytics/growthbook.js`
