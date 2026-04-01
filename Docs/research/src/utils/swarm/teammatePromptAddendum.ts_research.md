# teammatePromptAddendum.ts 深度研究文档

## 场景与职责

`teammatePromptAddendum.ts` 是 Claude Code 多代理集群（Agent Swarm）架构中的**Teammate 系统提示附加模块**，定义了附加到 teammate 系统提示末尾的特殊指令。

### 核心场景

当 Claude Code 作为 teammate 运行时，需要明确告知其：
1. **通信方式**：必须使用 `SendMessage` 工具与团队成员通信
2. **可见性约束**：纯文本响应对其他团队成员不可见
3. **角色定位**：用户主要与团队领导交互，teammate 通过任务系统和消息系统协调工作

### 职责边界

- 只提供系统提示附加文本
- 不处理提示的组装逻辑（由调用方处理）
- 作为常量字符串导出，无运行时逻辑

---

## 功能点目的

### 1. 通信指导

**内容**：
- 使用 `SendMessage` 工具与 `to: "<name>"` 发送给特定 teammate
- 使用 `SendMessage` 工具与 `to: "*"` 进行团队广播（谨慎使用）

### 2. 可见性约束

**内容**：
- 纯文本响应对其他团队成员不可见
- 必须使用 `SendMessage` 工具才能通信

### 3. 角色说明

**内容**：
- 用户主要与团队领导交互
- teammate 的工作通过任务系统和 teammate 消息系统协调

---

## 具体技术实现

### 数据结构

#### TEAMMATE_SYSTEM_PROMPT_ADDENDUM

```typescript
export const TEAMMATE_SYSTEM_PROMPT_ADDENDUM = `
# Agent Teammate Communication

IMPORTANT: You are running as an agent in a team. To communicate with anyone on your team:
- Use the SendMessage tool with \`to: "<name>"\` to send messages to specific teammates
- Use the SendMessage tool with \`to: "*"\` sparingly for team-wide broadcasts

Just writing a response in text is not visible to others on your team - you MUST use the SendMessage tool.

The user interacts primarily with the team lead. Your work is coordinated through the task system and teammate messaging.
`;
```

### 使用方式

该常量被附加到主系统提示的末尾，形成完整的 teammate 系统提示：

```
[主系统提示]

[TEAMMATE_SYSTEM_PROMPT_ADDENDUM]
```

---

## 关键代码路径与文件引用

### 核心导出

| 导出项 | 类型 | 用途 |
|--------|------|------|
| `TEAMMATE_SYSTEM_PROMPT_ADDENDUM` | string | teammate 系统提示附加文本 |

### 调用方文件

通过 Grep 搜索，该常量在代码库中被以下文件引用：

| 文件 | 用途 |
|------|------|
| 系统提示组装模块 | 附加到 teammate 的系统提示 |

### 依赖文件

该模块无外部依赖，为纯常量定义。

---

## 依赖与外部交互

### 模块依赖图

```
teammatePromptAddendum.ts
[无外部依赖]
```

### 与系统提示组装模块的交互

```typescript
// 伪代码示例
import { TEAMMATE_SYSTEM_PROMPT_ADDENDUM } from '../utils/swarm/teammatePromptAddendum.js';

function buildTeammateSystemPrompt(basePrompt: string): string {
  return `${basePrompt}\n${TEAMMATE_SYSTEM_PROMPT_ADDENDUM}`;
}
```

---

## 风险、边界与改进建议

### 已知风险

#### 1. 提示注入风险

**风险**：如果 `SendMessage` 工具的实现变化，提示中的指令可能过时。

**当前指令**：
```
- Use the SendMessage tool with `to: "<name>"` to send messages to specific teammates
- Use the SendMessage tool with `to: "*"` sparingly for team-wide broadcasts
```

**改进建议**：
```typescript
// 从 SendMessageTool 常量生成提示
import { SEND_MESSAGE_TOOL_NAME, BROADCAST_TARGET } from '../tools/SendMessageTool/constants.js';

export const TEAMMATE_SYSTEM_PROMPT_ADDENDUM = `
# Agent Teammate Communication

IMPORTANT: You are running as an agent in a team. To communicate with anyone on your team:
- Use the ${SEND_MESSAGE_TOOL_NAME} tool with \`to: "<name>"\` to send messages to specific teammates
- Use the ${SEND_MESSAGE_TOOL_NAME} tool with \`to: "${BROADCAST_TARGET}"\` sparingly for team-wide broadcasts

Just writing a response in text is not visible to others on your team - you MUST use the ${SEND_MESSAGE_TOOL_NAME} tool.

The user interacts primarily with the team lead. Your work is coordinated through the task system and teammate messaging.
`;
```

#### 2. 国际化缺失

**风险**：提示文本为硬编码英文，不支持多语言。

**改进建议**：
```typescript
// 支持基于用户语言的选择
const ADDENDUMS: Record<string, string> = {
  en: `
# Agent Teammate Communication

IMPORTANT: You are running as an agent in a team...
`,
  zh: `
# 代理团队成员通信

重要提示：您正在作为团队中的代理运行...
`,
  // ...
};

export function getTeammateSystemPromptAddendum(language: string = 'en'): string {
  return ADDENDUMS[language] ?? ADDENDUMS.en;
}
```

#### 3. 提示长度

**风险**：附加提示增加了系统提示长度，可能影响模型性能。

**当前长度**：约 350 字符

**改进建议**：
```typescript
// 提供精简版本
export const TEAMMATE_SYSTEM_PROMPT_ADDENDUM_SHORT = `
You are a teammate. Use SendMessage tool (to: "name" or to: "*") to communicate. Text responses are not visible to teammates.
`;

// 根据模型选择版本
export function getTeammateSystemPromptAddendum(modelId?: string): string {
  // 对于上下文窗口较小的模型使用精简版本
  if (modelId?.includes('haiku') || modelId?.includes('instant')) {
    return TEAMMATE_SYSTEM_PROMPT_ADDENDUM_SHORT;
  }
  return TEAMMATE_SYSTEM_PROMPT_ADDENDUM;
}
```

### 边界条件

| 场景 | 行为 |
|------|------|
| 非 teammate 实例 | 不应附加此提示 |
| 提示组装错误 | 可能导致 teammate 不知道如何使用 SendMessage |
| 工具名称变化 | 提示中的工具名称与实际不匹配 |

### 改进建议

#### 1. 动态提示生成

```typescript
// 根据团队配置动态生成提示
import { readTeamFile } from './teamHelpers.js';

export function generateTeammatePromptAddendum(
  teamName: string,
  agentName: string
): string {
  const teamFile = readTeamFile(teamName);
  const teammates = teamFile?.members
    .filter(m => m.name !== agentName)
    .map(m => m.name) ?? [];
  
  return `
# Agent Teammate Communication

You are part of team "${teamName}". Your teammates are: ${teammates.join(', ')}.

IMPORTANT: To communicate with anyone on your team:
- Use the SendMessage tool with \`to: "<name>"\` to send messages to specific teammates
- Use the SendMessage tool with \`to: "*"\` sparingly for team-wide broadcasts

Just writing a response in text is not visible to others on your team - you MUST use the SendMessage tool.

The user interacts primarily with the team lead. Your work is coordinated through the task system and teammate messaging.
`;
}
```

#### 2. 提示验证

```typescript
// 验证提示是否正确附加
export function validateTeammatePrompt(fullPrompt: string): boolean {
  const requiredPhrases = [
    'SendMessage tool',
    'not visible to others',
    'team lead',
  ];
  
  return requiredPhrases.every(phrase => 
    fullPrompt.includes(phrase)
  );
}
```

#### 3. A/B 测试支持

```typescript
// 支持不同提示变体的 A/B 测试
const ADDENDUM_VARIANTS = {
  control: TEAMMATE_SYSTEM_PROMPT_ADDENDUM,
  variant_a: `
# Team Communication

You are a teammate. Always use SendMessage to communicate with others.
`,
  variant_b: `
# Important: Team Communication Required

As a teammate, you MUST use the SendMessage tool for all team communication. Regular text is private.
`,
};

export function getTeammateSystemPromptAddendum(variant?: string): string {
  return ADDENDUM_VARIANTS[variant as keyof typeof ADDENDUM_VARIANTS] 
    ?? ADDENDUM_VARIANTS.control;
}
```

### 测试建议

1. **内容测试**：
   - 验证包含所有关键短语
   - 验证工具名称正确

2. **集成测试**：
   - 验证附加到系统提示后模型行为正确
   - 验证 teammate 正确使用 SendMessage 工具

3. **回归测试**：
   - 确保提示变化不会破坏现有功能
