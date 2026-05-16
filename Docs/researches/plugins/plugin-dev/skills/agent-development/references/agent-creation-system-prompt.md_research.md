# Agent Creation System Prompt 深度研究文档

## 场景与职责

### 定位与用途

`agent-creation-system-prompt.md` 是 Claude Code 内部 Agent 生成功能使用的**核心系统提示词模板**。该文件记录了 Claude Code 产品级 Agent 生成功能的完整提示词设计，是插件开发者利用 AI 辅助创建 Agent 的**权威参考实现**。

### 核心职责

1. **AI 辅助 Agent 生成**：为 Claude 提供结构化指令，将用户需求转换为完整的 Agent 配置
2. **标准化输出格式**：确保生成的 Agent 符合统一的 JSON 结构（identifier/whenToUse/systemPrompt）
3. **最佳实践嵌入**：将生产环境中验证有效的 Agent 设计模式编码到提示词中
4. **项目上下文感知**：指导 Claude 考虑 CLAUDE.md 中的项目特定规范

### 使用场景

| 场景 | 说明 |
|------|------|
| 插件开发 | 开发者为插件创建自定义 Agent 时 |
| Agent 迭代 | 基于现有 Agent 进行功能扩展或优化 |
| 团队协作 | 确保团队成员创建的 Agent 风格一致 |
| 新手引导 | 帮助不熟悉 Agent 结构的开发者快速上手 |

---

## 功能点目的

### 1. 六步 Agent 设计流程

该提示词定义了从需求到实现的六步转换流程：

```
用户描述需求
    ↓
1. Extract Core Intent（提取核心意图）
    ↓
2. Design Expert Persona（设计专家人设）
    ↓
3. Architect Comprehensive Instructions（构建完整指令）
    ↓
4. Optimize for Performance（性能优化）
    ↓
5. Create Identifier（创建标识符）
    ↓
6. Example agent descriptions（编写触发示例）
    ↓
输出 JSON 配置
```

### 2. 输出格式规范

强制要求输出为特定 JSON 结构：

```json
{
  "identifier": "agent-identifier",
  "whenToUse": "Use this agent when... <example>...</example>",
  "systemPrompt": "You are... **Your Core Responsibilities:**..."
}
```

**设计意图**：
- `identifier`：确保 Agent 名称符合技术约束（小写、连字符、长度限制）
- `whenToUse`：包含触发条件和示例，直接用于 Agent 文件的 `description` 字段
- `systemPrompt`：完整的系统提示词，直接用于 Agent 文件的 markdown 正文

### 3. 代码审查 Agent 的特殊处理

提示词中特别指出：
> "For agents that are meant to review code, you should assume that the user is asking to review recently written code and not the whole codebase"

**目的**：避免代码审查 Agent 过度扫描整个代码库，提高效率和针对性。

### 4. 主动触发示例生成

要求生成的示例必须展示**主动触发**模式：

```
<commentary>
Since a logical chunk of code was written and the task was completed, 
now use the code-review agent to review the code.
</commentary>
```

**目的**：培养 Agent 的主动性，不仅响应明确请求，还能在工作流适当时机自主调用。

---

## 具体技术实现

### 关键流程

#### 流程 1：JSON 输出生成流程

```
用户输入需求描述
    ↓
Claude 加载 agent-creation-system-prompt.md 作为系统提示词
    ↓
Claude 分析需求，执行六步设计流程
    ↓
生成符合格式的 JSON 对象
    ↓
用户将 JSON 转换为 Agent Markdown 文件
```

#### 流程 2：Agent 文件转换流程

```json
// JSON 输出
{
  "identifier": "pr-quality-reviewer",
  "whenToUse": "Use this agent when...",
  "systemPrompt": "You are..."
}
```

转换为：

```markdown
---
name: pr-quality-reviewer
description: Use this agent when...
model: inherit
color: blue
---

You are...
```

### 数据结构

#### 示例块结构（XML 风格标签）

```xml
<example>
Context: [场景描述]
user: "[用户消息]"
assistant: "[Claude 响应]"
<commentary>
[触发原因解释]
</commentary>
assistant: "[Agent 调用声明]"
</example>
```

#### 标识符命名约束

| 约束 | 规则 | 示例 |
|------|------|------|
| 字符集 | 小写字母、数字、连字符 | `code-reviewer` ✅ |
| 长度 | 通常 2-4 个词 | `api-docs-writer` ✅ |
| 禁用词 | helper, assistant 等泛化词 | `helper` ❌ |
| 格式 | 连字符连接 | `code_reviewer` ❌ |

### 协议与约定

#### 1. 系统提示词设计原则

```markdown
Key principles for your system prompts:
- Be specific rather than generic
- Include concrete examples when they would clarify behavior
- Balance comprehensiveness with clarity
- Ensure the agent has enough context to handle variations
- Make the agent proactive in seeking clarification
- Build in quality assurance and self-correction mechanisms
```

#### 2. 项目上下文集成协议

```markdown
**Important Context**: You may have access to project-specific instructions 
from CLAUDE.md files and other context that may include coding standards, 
project structure, and custom requirements.
```

**实现机制**：Claude Code 会自动将工作目录中的 CLAUDE.md 内容注入到上下文中，Agent 生成提示词指导 Claude 利用这些信息。

---

## 关键代码路径与文件引用

### 核心文件关系图

```
plugins/plugin-dev/
├── agents/
│   └── agent-creator.md          # 实际 Agent 实现（基于此提示词）
├── skills/agent-development/
│   ├── SKILL.md                  # 技能主文档
│   ├── references/
│   │   ├── agent-creation-system-prompt.md   # ← 本文件（参考模板）
│   │   ├── system-prompt-design.md           # 系统提示词设计模式
│   │   └── triggering-examples.md            # 触发示例最佳实践
│   ├── examples/
│   │   ├── agent-creation-prompt.md          # 使用示例
│   │   └── complete-agent-examples.md        # 完整示例
│   └── scripts/
│       └── validate-agent.sh     # Agent 验证脚本
```

### 调用关系

#### 被调用方（消费者）

| 文件 | 用途 |
|------|------|
| `SKILL.md` | 引用本文件作为 AI 辅助生成方法的参考 |
| `examples/agent-creation-prompt.md` | 基于此模板提供使用示例 |
| `agents/agent-creator.md` | Agent 实现，执行本提示词定义的任务 |

#### 调用方（依赖）

| 文件 | 依赖关系 |
|------|----------|
| `system-prompt-design.md` | 本文件引用的设计模式来源 |
| `triggering-examples.md` | 本文件引用的示例格式来源 |

### 关键代码片段

#### 1. Agent Creator Agent 实现（基于本提示词）

文件：`plugins/plugin-dev/agents/agent-creator.md`

```markdown
---
name: agent-creator
description: Use this agent when the user asks to "create an agent"...
model: sonnet
color: magenta
tools: ["Write", "Read"]
---

You are an elite AI agent architect...
```

**注意**：`agent-creator.md` 的 system prompt 与本文件内容高度一致，验证了本文件作为**生产级模板**的地位。

#### 2. 验证脚本中的约束检查

文件：`plugins/plugin-dev/skills/agent-development/scripts/validate-agent.sh`

```bash
# 标识符格式验证
if ! [[ "$NAME" =~ ^[a-zA-Z0-9][a-zA-Z0-9-]*[a-zA-Z0-9]$ ]]; then
  echo "❌ name must start/end with alphanumeric..."
fi

# 长度验证
if [ $name_length -lt 3 ]; then
  echo "❌ name too short (minimum 3 characters)"
fi

# 示例块检查
if ! echo "$DESCRIPTION" | grep -q '<example>'; then
  echo "⚠️  description should include <example> blocks"
fi
```

---

## 依赖与外部交互

### 内部依赖

| 依赖项 | 类型 | 说明 |
|--------|------|------|
| `system-prompt-design.md` | 设计参考 | 提供系统提示词结构模式 |
| `triggering-examples.md` | 格式参考 | 提供示例块格式规范 |
| `SKILL.md` | 使用指南 | 指导用户如何使用本提示词 |

### 外部依赖

| 依赖项 | 说明 |
|--------|------|
| Claude Code 平台 | 提供 Agent 加载和执行环境 |
| CLAUDE.md | 项目特定上下文（运行时注入） |

### 交互协议

#### 与 CLAUDE.md 的集成

```markdown
**Important Context**: You may have access to project-specific instructions 
from CLAUDE.md files...
```

**工作机制**：
1. Claude Code 自动检测工作目录中的 CLAUDE.md
2. 内容注入到对话上下文中
3. Agent 生成时参考其中的编码规范和项目结构

---

## 风险、边界与改进建议

### 已知风险

#### 风险 1：输出格式不一致

**表现**：Claude 可能输出非标准 JSON（如包含 markdown 代码块标记）。

**缓解措施**：
- 示例中明确要求 `"Return ONLY the JSON object, no other text"`
- 用户需手动验证 JSON 格式

#### 风险 2：项目上下文缺失

**表现**：生成的 Agent 未考虑项目特定规范。

**缓解措施**：
- 提示词中强调考虑 CLAUDE.md
- 建议用户在生成前确认 CLAUDE.md 存在且最新

#### 风险 3：触发条件过于宽泛

**表现**：生成的 `whenToUse` 描述可能导致 Agent 被错误触发。

**缓解措施**：
- 验证脚本检查 `"Use this agent when"` 模式
- 建议包含 2-4 个具体示例

### 边界条件

| 边界 | 限制 | 说明 |
|------|------|------|
| 标识符长度 | 3-50 字符 | 过短难以表达含义，过长难以记忆 |
| 系统提示词长度 | 建议 < 10,000 字符 | 避免性能下降 |
| 示例数量 | 建议 2-4 个 | 过少覆盖不全，过多信息冗余 |
| 描述长度 | 10-5,000 字符 | 确保足够详细但不冗长 |

### 改进建议

#### 建议 1：增加版本控制

```markdown
---
name: agent-creation-system-prompt
version: 1.0.0
last_updated: 2024-XX-XX
---
```

**理由**：便于追踪提示词变更对生成结果的影响。

#### 建议 2：添加输出示例验证

在提示词中增加：
```markdown
**Output Validation Checklist:**
- [ ] JSON is valid and parseable
- [ ] identifier matches naming conventions
- [ ] whenToUse includes at least 2 <example> blocks
- [ ] systemPrompt uses second person throughout
- [ ] systemPrompt includes responsibilities, process, and output format
```

#### 建议 3：支持多语言生成

当前提示词假设输出英文 Agent，可增加：
```markdown
Language: Generate the agent configuration in the same language as the user's request.
```

#### 建议 4：与验证脚本集成

建议 Claude Code 在生成后自动运行验证：
```bash
./scripts/validate-agent.sh agents/[identifier].md
```

### 生产环境注意事项

1. **提示词长度**：当前提示词约 3,500 字符，处于合理范围
2. **模型选择**：建议使用 Sonnet 或更高模型执行此提示词
3. **迭代优化**：根据实际生成结果定期更新提示词
4. **测试覆盖**：建议为常见 Agent 类型创建生成测试用例

---

## 总结

`agent-creation-system-prompt.md` 是 Claude Code Agent 生态系统的**核心基础设施文件**。它将生产环境中验证有效的 Agent 设计方法论编码为可复用的提示词模板，使插件开发者能够利用 AI 辅助快速创建高质量的自定义 Agent。

该文件的设计体现了以下核心原则：
- **结构化输出**：强制 JSON 格式确保一致性
- **上下文感知**：集成 CLAUDE.md 项目规范
- **最佳实践嵌入**：六步设计流程指导高质量输出
- **可验证性**：输出可直接转换为可验证的 Agent 文件
