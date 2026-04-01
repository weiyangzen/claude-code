# generateAgent.ts 深度研究文档

## 1. 场景与职责

### 1.1 核心场景
`generateAgent.ts` 是 Claude Code 中 **AI 驱动的 Agent 自动生成** 模块，利用 Claude API 根据用户自然语言描述自动生成完整的 Agent 配置。

主要应用场景：
1. **Agent 创建向导 - 生成步骤** (`GenerateStep.tsx`): 用户输入描述，AI 自动生成 Agent 配置
2. **统一建议系统** (`unifiedSuggestions.ts`): 为 Agent 建议提供数据支持

### 1.2 核心职责
- **自然语言理解**: 解析用户描述，提取 Agent 的核心意图
- **配置生成**: 自动生成 `identifier`（标识符）、`whenToUse`（使用时机）、`systemPrompt`（系统提示词）
- **上下文感知**: 结合项目 CLAUDE.md 等上下文生成更贴合的 Agent
- **记忆支持**: 根据配置自动添加 Agent Memory 相关指令

---

## 2. 功能点目的

### 2.1 AI 生成流程

```
用户描述 → 构建 Prompt → 调用 Claude API → 解析 JSON → 验证 → 返回配置
```

### 2.2 生成的 Agent 结构

```typescript
type GeneratedAgent = {
  identifier: string;      // 唯一标识符，如 "test-runner"
  whenToUse: string;       // 使用时机描述
  systemPrompt: string;    // 完整的系统提示词
};
```

### 2.3 Agent Memory 支持

当用户提及 "memory", "remember", "learn", "persist" 等关键词，或 Agent 适合跨会话积累知识时，自动在 systemPrompt 中添加记忆更新指令。

**示例**: 代码审查 Agent 的记忆指令：
```
"Update your agent memory as you discover code patterns, style conventions, 
common issues, and architectural decisions in this codebase."
```

---

## 3. 具体技术实现

### 3.1 核心数据结构

```typescript
type GeneratedAgent = {
  identifier: string;
  whenToUse: string;
  systemPrompt: string;
};
```

### 3.2 系统提示词设计

#### AGENT_CREATION_SYSTEM_PROMPT
这是一个精心设计的 "meta-prompt"，指导 Claude 如何创建 Agent：

**核心指令**：
1. **提取核心意图**: 识别 Agent 的根本目的、关键职责和成功标准
2. **设计专家人设**: 创建体现领域专业知识的角色设定
3. **架构全面指令**: 开发系统提示词，包括：
   - 行为边界和操作参数
   - 特定方法论和最佳实践
   - 边缘情况处理指导
   - 输出格式期望
4. **性能优化**: 包含决策框架、质量控制、工作流模式
5. **创建标识符**: 设计简洁、描述性的标识符（2-4个单词，小写+连字符）

**输出格式要求**（严格 JSON）：
```json
{
  "identifier": "...",
  "whenToUse": "...",
  "systemPrompt": "..."
}
```

#### AGENT_MEMORY_INSTRUCTIONS
当启用记忆功能时追加的指令：
- 指导 Agent 在发现领域特定信息时更新记忆
- 提供具体示例（代码审查、测试运行、架构设计等场景）

### 3.3 关键流程

#### 生成流程
```typescript
async function generateAgent(
  userPrompt: string,
  model: ModelName,
  existingIdentifiers: string[],
  abortSignal: AbortSignal
): Promise<GeneratedAgent>
```

```
1. 构建 Prompt
   ├── 用户描述: "Create an agent based on this request: '...'"
   ├── 已存在标识符列表（避免重复）
   └── 格式要求: "Return ONLY the JSON object"

2. 获取上下文
   ├── getUserContext() - 获取用户上下文（CLAUDE.md 等）
   └── prependUserContext() - 将上下文添加到消息

3. 组装系统提示词
   ├── AGENT_CREATION_SYSTEM_PROMPT
   └── 如果启用记忆: + AGENT_MEMORY_INSTRUCTIONS

4. 调用 API
   └── queryModelWithoutStreaming({
         messages,
         systemPrompt,
         thinkingConfig: { type: 'disabled' },  // 禁用思考模式
         tools: [],                               // 无工具
         options: { querySource: 'agent_creation' }
       })

5. 解析响应
   ├── 提取文本块
   ├── 尝试直接 JSON 解析
   ├── 失败则使用正则提取 JSON 对象
   └── 验证必需字段存在

6. 记录分析
   └── logEvent('tengu_agent_definition_generated', { agent_identifier })
```

### 3.4 关键算法

#### JSON 提取容错
```typescript
let parsed: GeneratedAgent;
try {
  parsed = jsonParse(responseText.trim());
} catch {
  // 如果直接解析失败，尝试提取 JSON 对象
  const jsonMatch = responseText.match(/\{[\s\S]*\}/);
  if (!jsonMatch) {
    throw new Error('No JSON object found in response');
  }
  parsed = jsonParse(jsonMatch[0]);
}
```

#### 字段验证
```typescript
if (!parsed.identifier || !parsed.whenToUse || !parsed.systemPrompt) {
  throw new Error('Invalid agent configuration generated');
}
```

---

## 4. 关键代码路径与文件引用

### 4.1 调用方

| 文件 | 调用函数 | 用途 |
|------|---------|------|
| `GenerateStep.tsx` | `generateAgent()` | 向导中生成 Agent |
| `unifiedSuggestions.ts` | 引用类型 | Agent 建议 |

### 4.2 依赖文件

| 文件 | 用途 |
|------|------|
| `src/services/api/claude.ts` | `queryModelWithoutStreaming()` - 非流式 API 调用 |
| `src/context.ts` | `getUserContext()` - 获取项目上下文 |
| `src/utils/api.ts` | `prependUserContext()` - 添加上下文到消息 |
| `src/utils/messages.ts` | `createUserMessage()`, `normalizeMessagesForAPI()` - 消息处理 |
| `src/utils/model/model.ts` | `ModelName` 类型 |
| `src/utils/systemPromptType.ts` | `asSystemPrompt()` - 系统提示词类型包装 |
| `src/utils/slowOperations.ts` | `jsonParse()` - JSON 解析 |
| `src/services/analytics/index.ts` | `logEvent()` - 分析日志 |
| `src/memdir/paths.ts` | `isAutoMemoryEnabled()` - 记忆功能开关 |
| `src/tools/AgentTool/constants.ts` | `AGENT_TOOL_NAME` - Agent 工具名 |
| `src/Tool.ts` | `getEmptyToolPermissionContext()` - 空权限上下文 |

### 4.3 常量定义

```typescript
// 来自 constants.ts
const AGENT_TOOL_NAME = 'Agent';  // 在示例中引用
```

---

## 5. 依赖与外部交互

### 5.1 API 调用

```typescript
const response = await queryModelWithoutStreaming({
  messages: normalizeMessagesForAPI(messagesWithContext),
  systemPrompt: asSystemPrompt([systemPrompt]),
  thinkingConfig: { type: 'disabled' as const },  // 禁用思考模式，提高速度
  tools: [],  // 生成过程不需要工具
  signal: abortSignal,
  options: {
    getToolPermissionContext: async () => getEmptyToolPermissionContext(),
    model,
    toolChoice: undefined,
    agents: [],
    isNonInteractiveSession: false,
    hasAppendSystemPrompt: false,
    querySource: 'agent_creation',  // 用于分析和追踪
    mcpTools: [],
  },
});
```

### 5.2 上下文集成

```typescript
// 获取项目上下文（CLAUDE.md 等）
const userContext = await getUserContext();

// 将上下文添加到用户消息前
const messagesWithContext = prependUserContext([userMessage], userContext);
```

### 5.3 记忆功能检测

```typescript
// 检查是否启用自动记忆
const systemPrompt = isAutoMemoryEnabled()
  ? AGENT_CREATION_SYSTEM_PROMPT + AGENT_MEMORY_INSTRUCTIONS
  : AGENT_CREATION_SYSTEM_PROMPT;
```

### 5.4 依赖关系图

```
generateAgent.ts
├── claude.ts (queryModelWithoutStreaming)
├── context.ts (getUserContext)
├── api.ts (prependUserContext)
├── messages.ts (createUserMessage, normalizeMessagesForAPI)
├── model/model.ts (ModelName)
├── systemPromptType.ts (asSystemPrompt)
├── slowOperations.ts (jsonParse)
├── analytics/index.ts (logEvent)
├── memdir/paths.ts (isAutoMemoryEnabled)
├── AgentTool/constants.ts (AGENT_TOOL_NAME)
└── Tool.ts (getEmptyToolPermissionContext)
```

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

#### 风险1: JSON 解析脆弱性
```typescript
const jsonMatch = responseText.match(/\{[\s\S]*\}/);
```
**问题**: 正则提取可能匹配到错误的 JSON 对象（如嵌套对象、代码块中的对象）。

**示例风险**:
```
Here's your agent:
```json
{ "identifier": "test" }
```
And here's an example output:
```json
{ "result": "success" }
```
```
正则会匹配到最后一个 `}`，包含两个 JSON 对象。

#### 风险2: 标识符冲突
虽然传入了 `existingIdentifiers`，但 AI 仍可能生成重复标识符（特别是并发请求时）。

#### 风险3: 提示词注入风险
用户输入直接拼接到 Prompt 中：
```typescript
const prompt = `Create an agent configuration based on this request: "${userPrompt}"...`;
```
**风险**: 恶意用户可能通过精心构造的输入影响 AI 行为。

#### 风险4: 长描述截断
如果用户输入极长的描述，可能超出模型的上下文限制。

### 6.2 边界情况

| 场景 | 处理行为 |
|------|---------|
| AI 返回非 JSON | 尝试正则提取，失败则抛错 |
| AI 返回部分字段 | 验证失败，抛错 "Invalid agent configuration generated" |
| 用户取消（AbortSignal） | 传播 AbortError |
| API 调用失败 | 抛出错误，由调用方处理 |
| 记忆功能禁用 | 不追加 AGENT_MEMORY_INSTRUCTIONS |
| 无现有标识符 | 不添加重复检查提示 |

### 6.3 改进建议

#### 建议1: 使用结构化输出（Structured Outputs）
使用 Claude 的 JSON 模式或工具调用确保输出格式正确：
```typescript
const response = await queryModelWithoutStreaming({
  // ...
  responseFormat: { type: 'json_object' },  // 强制 JSON 输出
});
```

#### 建议2: 更健壮的 JSON 提取
使用专门的 JSON 提取库：
```typescript
import { extractJson } from 'some-json-extractor';

const parsed = extractJson(responseText, {
  schema: GeneratedAgentSchema,
  multiple: false,  // 只取第一个匹配
});
```

#### 建议3: 输入验证和清理
```typescript
function sanitizeUserPrompt(prompt: string): string {
  // 限制长度
  const maxLength = 2000;
  if (prompt.length > maxLength) {
    prompt = prompt.slice(0, maxLength) + '...';
  }
  
  // 转义特殊字符
  return prompt.replace(/["\\]/g, '\\$&');
}
```

#### 建议4: 重试机制
```typescript
async function generateAgentWithRetry(..., maxRetries = 3): Promise<GeneratedAgent> {
  for (let i = 0; i < maxRetries; i++) {
    try {
      return await generateAgent(...);
    } catch (error) {
      if (i === maxRetries - 1) throw error;
      // 指数退避
      await delay(Math.pow(2, i) * 1000);
    }
  }
  throw new Error('Max retries exceeded');
}
```

#### 建议5: 标识符唯一性保证
```typescript
function makeIdentifierUnique(identifier: string, existing: string[]): string {
  let unique = identifier;
  let counter = 1;
  while (existing.includes(unique)) {
    unique = `${identifier}-${counter}`;
    counter++;
  }
  return unique;
}
```

#### 建议6: 生成过程可视化
当前生成是黑盒，可添加中间步骤输出：
```typescript
export type GenerationProgress = 
  | { stage: 'analyzing' }
  | { stage: 'designing_persona' }
  | { stage: 'writing_prompt' }
  | { stage: 'complete', result: GeneratedAgent };

export async function* generateAgentStreaming(...): AsyncGenerator<GenerationProgress> {
  // 使用流式 API，实时返回生成进度
}
```

### 6.4 测试建议

| 测试场景 | 验证点 |
|---------|--------|
| 恶意输入（Prompt Injection） | 输出不受影响 |
| 超长输入 | 正确处理，不崩溃 |
| 非 JSON 响应 | 优雅降级或报错 |
| 部分字段缺失 | 正确检测并报错 |
| 并发生成 | 标识符不冲突 |
| 取消信号 | 及时终止 |
| 记忆功能开关 | 正确追加/省略指令 |
| 不同模型 | 生成质量一致性 |
