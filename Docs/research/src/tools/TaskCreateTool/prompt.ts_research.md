# prompt.ts 研究文档

## 场景与职责

`prompt.ts` 是 TaskCreateTool 的提示文本定义文件，负责为 LLM 提供使用 TaskCreate 工具的详细指导。该文件通过动态生成提示文本，根据 Agent Swarms 功能的启用状态调整提示内容，确保模型在不同配置下都能正确理解工具的使用方法。

## 功能点目的

### 1. 工具使用指南
- 定义 `DESCRIPTION`：简短描述工具用途（`'Create a new task in the task list'`）
- 定义 `getPrompt()` 函数：返回详细的工具使用说明

### 2. 动态提示生成
- 根据 `isAgentSwarmsEnabled()` 状态动态调整提示内容
- 在 Agent Swarms 启用时添加团队协作相关的提示

### 3. 使用场景指导
- **何时使用**：复杂多步骤任务、计划模式、用户明确要求等
- **何时不使用**：单一简单任务、纯对话场景

### 4. 字段说明
- 解释 `subject`、`description`、`activeForm` 字段的含义和用法
- 说明任务默认状态为 `pending`

## 具体技术实现

### 数据结构

```typescript
// 静态描述
export const DESCRIPTION = 'Create a new task in the task list'

// 动态提示生成函数
export function getPrompt(): string {
  // 根据 Agent Swarms 状态调整
  const teammateContext = isAgentSwarmsEnabled() 
    ? ' and potentially assigned to teammates' 
    : ''
  const teammateTips = isAgentSwarmsEnabled() 
    ? '...teammate specific tips...' 
    : ''
  
  return `...full prompt text...`
}
```

### 提示内容结构

```
## When to Use This Tool
- 复杂多步骤任务（3+ 步骤）
- 非平凡复杂任务
- 计划模式
- 用户明确要求待办列表
- 用户提供多个任务
- 接收新指令后
- 开始任务前（标记为 in_progress）
- 完成任务后（添加后续任务）

## When NOT to Use This Tool
- 单一简单任务
- 平凡任务（< 3 步）
- 纯对话或信息性任务

## Task Fields
- subject: 简短、可操作的标题（命令式）
- description: 需要做什么
- activeForm: 进行时显示文本（可选）

## Tips
- 创建清晰、具体的主题
- 使用 TaskUpdate 设置依赖关系
- 先检查 TaskList 避免重复
```

### Agent Swarms 扩展内容

当 `isAgentSwarmsEnabled()` 返回 `true` 时：

1. ** teammateContext **：在 "When to Use" 中添加 `"and potentially assigned to teammates"`

2. ** teammateTips **：额外提示包括：
   - 描述中需包含足够细节，便于其他 Agent 理解和完成任务
   - 新任务创建为 `pending` 状态且无所有者
   - 使用 TaskUpdate 的 `owner` 参数分配任务

## 关键代码路径与文件引用

### 主要文件
- **当前文件**: `src/tools/TaskCreateTool/prompt.ts`

### 依赖文件
| 文件 | 用途 |
|------|------|
| `src/utils/agentSwarmsEnabled.ts` | `isAgentSwarmsEnabled()` 函数 |

### 被引用位置
| 文件 | 用途 |
|------|------|
| `src/tools/TaskCreateTool/TaskCreateTool.ts` | 导入 `DESCRIPTION` 和 `getPrompt` |

### 调用关系
```
TaskCreateTool
├── description() -> DESCRIPTION
└── prompt() -> getPrompt()
    └── isAgentSwarmsEnabled()
        ├── process.env.USER_TYPE === 'ant'
        ├── process.env.CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS
        ├── process.argv.includes('--agent-teams')
        └── GrowthBook killswitch 'tengu_amber_flint'
```

## 依赖与外部交互

### 外部依赖
```typescript
import { isAgentSwarmsEnabled } from '../../utils/agentSwarmsEnabled.js'
```

### isAgentSwarmsEnabled 逻辑
该函数决定 Agent Swarms 功能是否可用：

1. **Ant 构建**: 始终启用（`USER_TYPE === 'ant'`）
2. **外部构建**: 需要同时满足：
   - 通过 `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` 环境变量或 `--agent-teams` 命令行标志显式启用
   - GrowthBook 功能开关 `tengu_amber_flint` 未禁用（killswitch）

### 提示文本使用流程
1. LLM 请求工具列表时，调用 `TaskCreateTool.description()`
2. 工具被选中使用时，调用 `TaskCreateTool.prompt()` 获取详细说明
3. 提示文本注入到 LLM 上下文中，指导其正确使用工具

## 风险、边界与改进建议

### 潜在风险

1. **提示文本过长**
   - 风险：详细提示可能占用过多上下文窗口
   - 当前：约 2.7KB，相对可控
   - 缓解：使用 `shouldDefer: true` 延迟加载

2. **功能开关不一致**
   - 风险：`isAgentSwarmsEnabled()` 在提示生成和实际运行时可能返回不同值
   - 场景：GrowthBook 开关在会话期间变更
   - 缓解：提示变更不会破坏功能，仅影响指导准确性

3. **国际化缺失**
   - 风险：提示文本仅支持英文
   - 影响：非英语用户可能理解困难
   - 现状：Claude Code 整体为英文界面，当前可接受

### 边界条件

1. **Agent Swarms 状态变更**
   - 边界：用户在会话期间启用/禁用 Agent Swarms
   - 行为：提示文本在工具调用时动态生成，反映当前状态

2. **提示缓存**
   - 边界：某些实现可能缓存提示文本
   - 风险：状态变更后仍显示旧提示
   - 当前：`getPrompt()` 每次调用都重新评估状态

### 改进建议

1. **提示版本控制**
   ```typescript
   export const PROMPT_VERSION = '1.0.0'
   
   export function getPrompt(): string {
     return `...prompt text... (v${PROMPT_VERSION})`
   }
   ```
   - 用途：便于追踪提示变更对模型行为的影响

2. **A/B 测试支持**
   ```typescript
   export function getPrompt(variant?: 'control' | 'treatment'): string {
     switch(variant) {
       case 'treatment': return getTreatmentPrompt()
       default: return getControlPrompt()
     }
   }
   ```
   - 用途：测试不同提示对工具使用准确性的影响

3. **动态示例注入**
   ```typescript
   export function getPrompt(context?: { recentTasks?: string[] }): string {
     const examples = context?.recentTasks 
       ? generateExamplesFromTasks(context.recentTasks)
       : DEFAULT_EXAMPLES
     return `${BASE_PROMPT}\n\n## Examples\n${examples}`
   }
   ```
   - 用途：根据上下文提供相关示例

4. **分层提示结构**
   ```typescript
   export const PROMPT_SECTIONS = {
     description: '...',
     whenToUse: '...',
     whenNotToUse: '...',
     fields: '...',
     tips: '...'
   } as const
   
   export function getPrompt(options?: { includeSections?: (keyof typeof PROMPT_SECTIONS)[] }): string {
     // 允许选择性包含章节
   }
   ```
   - 用途：支持根据上下文裁剪提示长度

5. **提示效果度量**
   - 建议：添加遥测追踪提示版本与工具使用准确性的关联
   - 实现：在工具调用结果中记录使用的提示版本
   - 分析：识别提示中哪些指导最有效

6. **多语言支持准备**
   ```typescript
   // prompt.ts
   import { getLocalizedPrompt } from '../../i18n/prompts.js'
   
   export function getPrompt(locale?: string): string {
     return getLocalizedPrompt('TaskCreate', locale, {
       teammateContext: isAgentSwarmsEnabled() ? '...' : ''
     })
   }
   ```
   - 用途：为未来国际化做准备

### 相关提示文件参考

Claude Code 中其他工具的提示组织方式：
- `src/tools/TaskUpdateTool/prompt.ts` - 任务更新工具
- `src/tools/TaskListTool/prompt.ts` - 任务列表工具
- `src/tools/TaskCompleteTool/prompt.ts` - 任务完成工具

共同模式：
- 使用 `DESCRIPTION` + `getPrompt()` 导出
- 动态内容通过函数参数或环境检查实现
- 提示文本使用 Markdown 格式
