# prompt.ts 研究文档

## 场景与职责

`prompt.ts` 是 `TeamDeleteTool` 的提示词定义文件，负责为 LLM（大语言模型）提供工具使用指南。该文件遵循 Claude Code CLI 的工具模块架构模式，将提示词逻辑与工具业务逻辑分离。

### 核心职责
1. **工具描述**：向 LLM 解释 TeamDelete 工具的用途和使用场景
2. **操作说明**：详细说明工具执行的具体操作
3. **前置条件**：明确工具调用的前提要求
4. **使用时机**：指导 LLM 何时应该调用此工具

## 功能点目的

### 1. 工具功能说明
- 说明工具用于在 swarm 工作完成后移除团队和任务目录
- 列出具体的清理操作：团队目录、任务目录、团队上下文

### 2. 安全检查强调
- **重要警告**：TeamDelete 会在团队仍有活跃成员时失败
- 指导 LLM 先优雅地终止队友，再调用 TeamDelete
- 强调执行顺序：先 `requestShutdown`，后 `TeamDelete`

### 3. 自动团队名解析
- 说明团队名自动从当前会话的团队上下文中确定
- LLM 无需（也不能）手动指定团队名

## 具体技术实现

### 函数定义

```typescript
export function getPrompt(): string {
  return `
# TeamDelete

Remove team and task directories when the swarm work is complete.

This operation:
- Removes the team directory (\`~/.claude/teams/{team-name}/\`)
- Removes the task directory (\`~/.claude/tasks/{team-name}/\`)
- Clears team context from the current session

**IMPORTANT**: TeamDelete will fail if the team still has active members. Gracefully terminate teammates first, then call TeamDelete after all teammates have shut down.

Use this when all teammates have finished their work and you want to clean up the team resources. The team name is automatically determined from the current session's team context.
`.trim()
}
```

### 提示词结构

```markdown
# TeamDelete
[工具名称标题]

Remove team and task directories when the swarm work is complete.
[一句话功能概述]

This operation:
- Removes the team directory (`~/.claude/teams/{team-name}/`)
- Removes the task directory (`~/.claude/tasks/{team-name}/`)
- Clears team context from the current session
[具体操作列表]

**IMPORTANT**: TeamDelete will fail if the team still has active members...
[重要警告和前置条件]

Use this when all teammates have finished their work...
[使用时机指导]
```

### 路径转义

```typescript
// 使用反斜杠转义反引号，确保在模板字符串中正确显示
`\`~/.claude/teams/{team-name}/\``
// 渲染结果: `~/.claude/teams/{team-name}/`
```

## 关键代码路径与文件引用

### 直接依赖

无外部导入，纯字符串生成函数。

### 被调用方

| 路径 | 用途 |
|------|------|
| `./TeamDeleteTool.ts` | 在 `buildTool` 的 `prompt()` 方法中调用 |

### 调用链

```
TeamDeleteTool.ts
  ├─ import { getPrompt } from './prompt.js'
  └─ buildTool({
       async prompt() {
         return getPrompt()
       },
     })
```

### 工具注册流程

```
getPrompt() 
  → TeamDeleteTool.prompt()
    → buildTool()
      → 注册到工具系统
        → 在 LLM 系统提示中展示
```

## 依赖与外部交互

### 无外部依赖

该文件是一个纯提示词生成文件：
- 无 `import` 语句
- 无运行时依赖
- 不依赖应用状态或配置

### 硬编码路径

提示词中硬编码了以下路径模式：
- `~/.claude/teams/{team-name}/`
- `~/.claude/tasks/{team-name}/`

这些路径对应实际实现中的：
```typescript
// teamHelpers.ts
getTeamDir(teamName)      // ~/.claude/teams/{sanitized-name}/
getTasksDir(teamName)     // ~/.claude/tasks/{sanitized-name}/
```

## 风险、边界与改进建议

### 风险点

1. **路径不一致**
   - 提示词中的路径是描述性的，可能与实际代码实现不完全一致
   - 实际路径使用 `sanitizeName()` 处理团队名
   - 建议：确保提示词描述与实际行为一致

2. **提示词过长**
   - 当前提示词简洁，但可能缺少某些边界情况的说明
   - 例如：重复调用、部分失败场景

3. **国际化缺失**
   - 提示词为英文硬编码
   - 不支持多语言界面

### 边界情况

1. **空团队名**
   - 提示词说明团队名自动确定
   - 但未说明如果 teamContext 为空会发生什么

2. **清理失败**
   - 提示词提到会失败（如果有活跃成员）
   - 但未说明其他失败场景（权限不足、磁盘错误等）

3. **并发调用**
   - 未说明如果多个 TeamDelete 同时调用的行为

### 改进建议

1. **添加错误场景说明**
   ```typescript
   export function getPrompt(): string {
     return `
   # TeamDelete
   
   Remove team and task directories when the swarm work is complete.
   
   This operation:
   - Removes the team directory (\`~/.claude/teams/{team-name}/\`)
   - Removes the task directory (\`~/.claude/tasks/{team-name}/\`)
   - Clears team context from the current session
   
   **IMPORTANT**: TeamDelete will fail if:
   - The team still has active members (use requestShutdown first)
   - No team context exists in the current session
   
   **Returns**: 
   - success: true if cleanup completed
   - message: Description of what was cleaned up
   - team_name: The name of the cleaned team (if any)
   
   Use this when all teammates have finished their work...
   `.trim()
   }
   ```

2. **动态路径生成**
   ```typescript
   import { getTeamsDir, getTasksDir } from '../../utils/envUtils.js'
   
   export function getPrompt(): string {
     const teamsDir = getTeamsDir()
     const tasksDir = getTasksDir('{team-name}')
     return `
   # TeamDelete
   
   This operation:
   - Removes the team directory (\`${teamsDir}/{team-name}/\`)
   - Removes the task directory (\`${tasksDir}\`)
   ...
   `.trim()
   }
   ```

3. **版本控制**
   - 添加提示词版本号
   - 便于追踪提示词变更对 LLM 行为的影响
   ```typescript
   export const PROMPT_VERSION = '1.0.0'
   ```

4. **A/B 测试支持**
   - 考虑支持多个提示词变体
   - 便于测试不同提示词对 LLM 使用模式的影响

5. **添加使用示例**
   ```typescript
   export function getPrompt(): string {
     return `
   # TeamDelete
   
   ...
   
   **Example workflow:**
   1. Call requestShutdown for each teammate
   2. Wait for all teammates to complete their current tasks
   3. Call TeamDelete to clean up resources
   `.trim()
   }
   ```
