# constants.ts 深度研究文档

## 场景与职责

`constants.ts` 是 Claude Code 中 Agent 工具的基础常量定义模块。虽然代码量很小（仅 12 行），但它是整个 Agent 工具系统的命名基础，定义了工具名称、向后兼容性标识以及一次性代理类型集合等关键常量。

该模块的核心职责：
1. **工具命名标准化**：定义 Agent 工具的正式名称和遗留名称
2. **向后兼容性**：维护遗留名称以支持旧版本会话恢复
3. **一次性代理标识**：定义不需要返回完整结果的轻量级代理类型

## 功能点目的

### 1. 工具名称常量
- `AGENT_TOOL_NAME = 'Agent'`：Agent 工具的正式名称
- `LEGACY_AGENT_TOOL_NAME = 'Task'`：遗留名称，用于向后兼容

### 2. 验证代理类型
- `VERIFICATION_AGENT_TYPE = 'verification'`：验证代理的类型标识

### 3. 一次性代理类型集合
- `ONE_SHOT_BUILTIN_AGENT_TYPES`：包含 `Explore` 和 `Plan` 两种代理类型
- 这些代理运行一次后返回报告，父代理不会继续发送消息
- 用于节省 token（约 135 字符 × 3400 万次运行/周）

## 具体技术实现

### 常量定义

```typescript
// Agent 工具的正式名称
export const AGENT_TOOL_NAME = 'Agent'

// 遗留名称，用于向后兼容（权限规则、hooks、恢复的会话）
export const LEGACY_AGENT_TOOL_NAME = 'Task'

// 验证代理类型
export const VERIFICATION_AGENT_TYPE = 'verification'

// 一次性内置代理类型集合
// 这些代理运行一次后返回报告，父代理不会继续发送消息
// 跳过 agentId/SendMessage/usage 尾部信息以节省 token
export const ONE_SHOT_BUILTIN_AGENT_TYPES: ReadonlySet<string> = new Set([
  'Explore',
  'Plan',
])
```

### 设计决策

**1. 为什么需要 LEGACY_AGENT_TOOL_NAME？**
- 旧版本中使用 `Task` 作为工具名称
- 权限规则、hooks 和持久化会话可能仍引用旧名称
- 保持向后兼容，确保旧会话能够正确恢复

**2. 为什么 ONE_SHOT_BUILTIN_AGENT_TYPES 是 ReadonlySet？**
- 防止运行时修改
- 使用 Set 提供 O(1) 的查找性能
- 明确表示这是不可变的配置

**3. 为什么 Explore 和 Plan 是一次性的？**
- 它们是只读搜索代理
- 父代理接收报告后不需要继续交互
- 跳过结果尾部可显著节省 token（每周节省约 5-15 Gtok）

## 依赖与外部交互

### 依赖关系

该模块是叶子模块，**不依赖任何其他模块**，只导出常量。

### 被调用方

通过 Grep 搜索，该模块被广泛使用：

| 文件路径 | 用途 |
|---------|------|
| `src/tools/AgentTool/agentToolUtils.ts` | 工具过滤和解析 |
| `src/tools/AgentTool/prompt.ts` | 提示构建 |
| `src/tools/AgentTool/runAgent.ts` | 代理运行 |
| `src/constants/tools.ts` | 工具常量定义 |
| `src/tools/ExitPlanModeTool/ExitPlanModeV2Tool.ts` | 计划模式工具 |
| `src/tools/TaskUpdateTool/TaskUpdateTool.ts` | 任务更新工具 |
| `src/utils/permissions/permissions.ts` | 权限处理 |
| `src/utils/attachments.ts` | 附件处理 |
| `src/utils/messages.ts` | 消息处理 |

### 使用示例

```typescript
// 检查是否为一次性代理
import { ONE_SHOT_BUILTIN_AGENT_TYPES } from './constants.js'

function shouldSkipResultTrailer(agentType: string): boolean {
  return ONE_SHOT_BUILTIN_AGENT_TYPES.has(agentType)
}

// 工具名称匹配
import { AGENT_TOOL_NAME, LEGACY_AGENT_TOOL_NAME } from './constants.js'

function isAgentTool(name: string): boolean {
  return name === AGENT_TOOL_NAME || name === LEGACY_AGENT_TOOL_NAME
}
```

## 风险、边界与改进建议

### 已知风险

1. **命名冲突**：`AGENT_TOOL_NAME` 是通用的 "Agent"，可能与其他工具冲突
2. **遗留名称维护**：`LEGACY_AGENT_TOOL_NAME` 需要长期维护，增加技术债务
3. **硬编码代理类型**：`ONE_SHOT_BUILTIN_AGENT_TYPES` 中的代理类型名是硬编码的，如果代理重命名会失效

### 边界情况

1. **大小写敏感**：`ONE_SHOT_BUILTIN_AGENT_TYPES` 使用精确匹配，大小写敏感
2. **空字符串**：未对空字符串进行特殊处理
3. **代理类型扩展**：添加新的一次性代理需要修改此文件

### 改进建议

1. **集中配置**：
   - 将 `ONE_SHOT_BUILTIN_AGENT_TYPES` 移到代理定义文件中
   - 通过代理定义的属性（如 `isOneShot: true`）来标识

2. **命名空间**：
   - 考虑为工具名称添加命名空间前缀，避免冲突
   - 例如：`claude:Agent` 或 `cc:Agent`

3. **版本管理**：
   - 添加版本号常量，用于会话兼容性检查
   - 定义最小兼容版本

4. **文档化**：
   - 添加更多注释说明每个常量的用途和历史背景
   - 记录遗留名称的弃用计划

5. **类型安全**：
   - 考虑使用字符串字面量类型替代普通 string
   - 例如：`export type AgentToolName = 'Agent' | 'Task'`

### 代码组织建议

1. **分离关注点**：
   - 将工具名称常量移到更通用的 `src/constants/toolNames.ts`
   - 将一次性代理标识移到代理定义模块

2. **配置化**：
   - 考虑将这些常量改为可从配置文件覆盖
   - 支持不同部署环境的定制化

### 向后兼容策略

1. **遗留名称移除计划**：
   - 制定 `LEGACY_AGENT_TOOL_NAME` 的移除时间表
   - 添加使用统计，确认无旧会话后再移除

2. **迁移工具**：
   - 提供工具将旧会话中的 `Task` 名称迁移到 `Agent`
   - 在会话恢复时自动转换
