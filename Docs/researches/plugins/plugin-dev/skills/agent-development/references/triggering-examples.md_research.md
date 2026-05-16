# Agent Triggering Examples 研究文档

## 场景与职责

### 文件定位
本文件位于 `plugins/plugin-dev/skills/agent-development/references/triggering-examples.md`，是 Claude Code 插件开发工具包中 Agent 开发技能的关键参考文档。

### 核心场景
该文档定义了在 Agent 描述（description）中编写 `<example>` 块的标准格式和最佳实践，用于确保 Agent 能够被可靠地触发。这是 Agent 开发中最关键但最容易出错的环节之一。

### 职责范围
1. **触发机制定义**: 定义 Agent 触发的工作原理和关键要素
2. **示例格式规范**: 提供标准化的 `<example>` 块格式
3. **触发类型分类**: 定义四种触发类型（显式请求、主动触发、隐式请求、工具使用模式）
4. **数量指导**: 提供示例数量的最小、推荐和最大建议
5. **调试指南**: 提供触发问题的诊断和修复方法

---

## 功能点目的

### 1. 示例块格式标准化

**标准格式**:
```markdown
<example>
Context: [描述情境 - 导致此交互的背景]
user: "[确切的用户消息或请求]"
assistant: "[Claude 在触发前应如何响应]"
<commentary>
[解释为什么在此场景应触发此 Agent]
</commentary>
assistant: "[Claude 如何触发 Agent - 通常是 '我将使用 [agent-name] Agent...']"
</example>
```

### 2. 优秀示例的解剖

#### Context（情境）
**目的**: 设定场景 - 用户消息之前发生了什么

**好的情境**:
```
Context: User just implemented a new authentication feature
Context: User has created a PR and wants it reviewed
Context: User is debugging a test failure
Context: After writing several functions without documentation
```

**差的情境**:
```
Context: User needs help (太模糊)
Context: Normal usage (不具体)
```

#### User Message（用户消息）
**目的**: 展示应该触发 Agent 的确切措辞

**好的用户消息**:
```
user: "I've added the OAuth flow, can you check it?"
user: "Review PR #123"
user: "Why is this test failing?"
user: "Add docs for these functions"
```

**措辞变化**: 为相同意图包含多个不同措辞的示例：
```markdown
Example 1: user: "Review my code"
Example 2: user: "Can you check this implementation?"
Example 3: user: "Look over my changes"
```

#### Assistant Response - Before Triggering（触发前响应）
**目的**: 展示 Claude 在启动 Agent 前的回应

**好的响应**:
```
assistant: "I'll analyze your OAuth implementation."
assistant: "Let me review that PR for you."
assistant: "I'll investigate the test failure."
```

**主动触发示例**:
```markdown
assistant: "Great! Now let me review the code quality."
<commentary>
Code was just written, proactively trigger review agent.
</commentary>
```

#### Commentary（注释）
**目的**: 解释推理 - 为什么应该在此场景触发此 Agent

**好的注释**:
```markdown
<commentary>
User explicitly requested code review, trigger the code-reviewer agent.
</commentary>

<commentary>
After code implementation, proactively use review agent to check quality.
</commentary>

<commentary>
PR analysis request matches pr-analyzer agent's expertise.
</commentary>
```

**包含决策逻辑**:
```markdown
<commentary>
User wrote tests (Test tool used). The test-analyzer agent should check
test quality and coverage before continuing.
</commentary>
```

#### Assistant Response - Triggering（触发响应）
**目的**: 展示 Claude 如何调用 Agent

**标准模式**:
```
assistant: "I'll use the [agent-name] agent to [what it will do]."
```

**示例**:
```
assistant: "I'll use the code-reviewer agent to analyze the changes."
assistant: "Let me use the test-generator agent to create comprehensive tests."
assistant: "I'll use the security-analyzer agent to check for vulnerabilities."
```

### 3. 触发类型分类

#### Type 1: 显式请求 (Explicit Request)
用户直接要求 Agent 的功能：
```markdown
<example>
Context: User explicitly requests security analysis
user: "Check my code for security vulnerabilities"
assistant: "I'll use the security-analyzer agent to perform a thorough security review."
<commentary>
Direct security analysis request triggers the security-analyzer agent.
</commentary>
</example>
```

#### Type 2: 主动触发 (Proactive Triggering)
在相关工作完成后自动触发，无需显式请求：
```markdown
<example>
Context: User just implemented database queries
user: "I've added the user search functionality"
assistant: "Great! Let me review it for security."
<commentary>
Database code written. Proactively trigger security-analyzer to check
for SQL injection and other database security issues.
</commentary>
assistant: "I'll use the security-analyzer agent to check the database code."
</example>
```

#### Type 3: 隐式请求 (Implicit Request)
用户暗示需求但未直接说明：
```markdown
<example>
Context: User mentions code is hard to understand
user: "This code is confusing and hard to follow"
assistant: "I can help improve the code clarity."
<commentary>
User indicates complexity issues. Trigger code-simplifier agent to refactor
for better readability.
</commentary>
assistant: "I'll use the code-simplifier agent to suggest improvements."
</example>
```

#### Type 4: 工具使用模式 (Tool Usage Pattern)
基于先前工具使用触发 Agent：
```markdown
<example>
Context: User made multiple edits to test files
user: "I've updated all the tests"
assistant: "Great! Let me verify test quality."
<commentary>
Multiple Edit tools used on test files. Proactively trigger test-quality-analyzer
to ensure tests follow best practices.
</commentary>
assistant: "I'll use the test-quality-analyzer agent to review the tests."
</example>
```

### 4. 多示例策略

#### 覆盖不同措辞
```markdown
<example>
user: "Review my code"
[...]
</example>

<example>
user: "Can you check my implementation?"
[...]
</example>

<example>
user: "Look over these changes"
[...]
</example>
```

#### 覆盖主动和被动触发
```markdown
<example>
Context: User explicitly requests review
user: "Review my code for issues"
[...]
</example>

<example>
Context: After user writes code
user: "I've implemented the feature"
assistant: "Great! Now let me review it."
<commentary>
Code written, proactively review.
</commentary>
[...]
</example>
```

#### 覆盖边界情况
```markdown
<example>
Context: Typical usage
user: "Check my PR"
[...]
</example>

<example>
Context: Large PR that needs thorough analysis
user: "This is a big PR, can you do a deep review?"
[...]
<commentary>
Large PR mentioned, use thorough analysis mode.
</commentary>
</example>
```

### 5. 示例数量指导

| 级别 | 数量 | 覆盖内容 |
|------|------|----------|
| 最小 | 2 个示例 | 1. 显式请求 2. 一种变化或主动触发 |
| 推荐 | 3-4 个示例 | 1. 显式请求 2. 隐式请求 3. 主动触发 4. 边界情况 |
| 最大 | 6 个示例 | 超过 6 个会使描述过长 |

---

## 具体技术实现

### 常见错误与修正

#### 错误 1: 缺失情境
```markdown
❌ <example>
user: "Review my code"
assistant: "I'll use the code-reviewer agent."
</example>

✅ <example>
Context: User just finished implementing authentication
user: "Review my code"
assistant: "I'll use the code-reviewer agent to analyze the auth implementation."
<commentary>
Auth code written, review for security and best practices.
</commentary>
</example>
```

#### 错误 2: 无注释
```markdown
❌ <example>
Context: User requests review
user: "Check my changes"
assistant: "I'll use the reviewer agent."
</example>

✅ <example>
Context: User requests review after making changes
user: "Check my changes"
assistant: "I'll use the code-reviewer agent to analyze the modifications."
<commentary>
Code review request triggers agent to check quality, security, and adherence to standards.
</commentary>
</example>
```

#### 错误 3: Agent 直接响应
```markdown
❌ <example>
user: "Review my code"
assistant: "I found the following issues: [lists issues]"
</example>

✅ <example>
user: "Review my code"
assistant: "I'll use the code-reviewer agent to perform the review."
<commentary>
Review request triggers the agent.
</commentary>
</example>
```

### 模板库

#### 代码审查 Agent 模板
```markdown
<example>
Context: User just implemented a new feature
user: "I've added the user registration feature"
assistant: "Great! Let me review the code quality."
<commentary>
Code written, proactively trigger code-reviewer agent.
</commentary>
assistant: "I'll use the code-reviewer agent to analyze the implementation."
</example>

<example>
Context: User explicitly requests review
user: "Can you review my changes for issues?"
assistant: "I'll use the code-reviewer agent to perform a thorough review."
<commentary>
Explicit review request triggers the agent.
</commentary>
</example>
```

#### 测试生成 Agent 模板
```markdown
<example>
Context: User implemented functions without tests
user: "I've added the data processing functions"
assistant: "Great! Let me generate tests for these."
<commentary>
New code without tests, proactively trigger test-generator agent.
</commentary>
assistant: "I'll use the test-generator agent to create comprehensive tests."
</example>

<example>
Context: User explicitly asks for tests
user: "Generate tests for my code"
assistant: "I'll use the test-generator agent to create a complete test suite."
<commentary>
Direct test generation request triggers the agent.
</commentary>
</example>
```

#### 文档 Agent 模板
```markdown
<example>
Context: User added new API endpoints
user: "I've implemented the REST API endpoints"
assistant: "I'll document these endpoints."
<commentary>
New API code, proactively generate documentation.
</commentary>
assistant: "I'll use the api-docs-writer agent to create comprehensive documentation."
</example>

<example>
Context: User requests documentation
user: "Write docs for this API"
assistant: "I'll use the api-docs-writer agent to generate clear documentation."
<commentary>
Documentation request triggers the agent.
</commentary>
</example>
```

#### 验证 Agent 模板
```markdown
<example>
Context: User ready to commit changes
user: "I'm ready to commit this code"
assistant: "Let me validate it first."
<commentary>
Before commit, proactively validate with validation-agent.
</commentary>
assistant: "I'll use the code-validator agent to check for issues."
</example>

<example>
Context: User asks for validation
user: "Validate my implementation"
assistant: "I'll use the code-validator agent to verify correctness."
<commentary>
Explicit validation request triggers the agent.
</commentary>
</example>
```

### 调试触发问题

#### Agent 不触发
**检查清单**:
1. 示例是否包含用户消息中的相关关键词
2. 情境是否匹配实际使用场景
3. 注释是否清晰解释触发逻辑
4. 助手是否展示使用 Agent 工具

**修复**: 添加覆盖不同措辞的更多示例

#### Agent 过度触发
**检查清单**:
1. 示例是否过于宽泛或通用
2. 触发条件是否与其他 Agent 重叠
3. 注释是否未区分何时不使用

**修复**: 使示例更具体，添加负面示例

#### Agent 在错误场景触发
**检查清单**:
1. 示例是否与实际预期用途不匹配
2. 注释是否建议不适当的触发

**修复**: 修改示例以仅展示正确的触发场景

---

## 关键代码路径与文件引用

### 本文件在系统中的位置
```
plugins/plugin-dev/
├── skills/agent-development/
│   ├── SKILL.md                           # 引用本文件作为触发示例参考
│   ├── references/
│   │   ├── agent-creation-system-prompt.md # 要求生成的 Agent 包含示例
│   │   ├── system-prompt-design.md        # 系统提示词设计
│   │   └── triggering-examples.md         # ← 本文件
│   ├── examples/
│   │   ├── agent-creation-prompt.md       # 展示如何生成包含示例的 Agent
│   │   └── complete-agent-examples.md     # 包含完整示例的应用
│   └── scripts/
│       └── validate-agent.sh              # 验证示例块存在
```

### 文档依赖关系

**上游依赖**:
- `system-prompt-design.md` - 系统提示词结构与触发示例协同工作

**下游引用**:
- `SKILL.md` - 在 "description" 和 "Creating Agents" 章节引用
- `agent-creation-system-prompt.md` - 要求生成的 Agent 包含 2-4 个示例
- `complete-agent-examples.md` - 每个示例 Agent 都包含多个触发示例

### 验证脚本检查点

`validate-agent.sh` 中与触发示例相关的检查：
```bash
# Check for example blocks
if ! echo "$DESCRIPTION" | grep -q '<example>'; then
  echo "⚠️  description should include <example> blocks for triggering"
  ((warning_count++))
fi

# Check for "Use this agent when" pattern
if ! echo "$DESCRIPTION" | grep -qi 'use this agent when'; then
  echo "⚠️  description should start with 'Use this agent when...'"
  ((warning_count++))
fi
```

---

## 依赖与外部交互

### 内部依赖

| 依赖项 | 类型 | 说明 |
|--------|------|------|
| SKILL.md | 被引用 | 核心技能文档整合本文件内容 |
| agent-creation-system-prompt.md | 协同 | 生成 Agent 时要求包含示例 |
| system-prompt-design.md | 协同 | 系统提示词与触发条件共同定义 Agent |
| validate-agent.sh | 配套 | 验证示例块存在性 |

### 与 Claude Code 触发机制的交互

**触发决策流程**:
1. Claude Code 加载所有插件的 Agent
2. 解析每个 Agent 的 `description` 字段
3. 用户输入时，匹配 description 中的触发条件
4. `<example>` 块帮助 Claude 理解何时触发
5. 匹配成功时，启动对应 Agent

**关键交互点**:
- `description` 字段是触发决策的主要依据
- `<example>` 块提供具体的匹配模式
- `commentary` 帮助 Claude 理解触发逻辑

### 与 Agent 创建流程的集成

**AI 辅助生成时的集成**:
1. 用户使用 `agent-creation-system-prompt.md` 生成 Agent
2. 提示词要求生成 2-4 个 `<example>` 块
3. 生成的示例应遵循本文件的格式规范
4. 使用 `validate-agent.sh` 验证示例存在

---

## 风险、边界与改进建议

### 潜在风险

#### 1. 触发歧义
**风险**: 多个 Agent 的触发条件重叠，导致错误的 Agent 被触发。

**示例**:
```markdown
Agent A: "Use when user asks to review code"
Agent B: "Use when user asks to check code quality"
# 用户说: "Check my code"
# 可能同时匹配 A 和 B
```

**缓解策略**:
- 使用更具体的触发条件
- 在 commentary 中明确区分使用场景
- 考虑 Agent 的优先级或互斥性

#### 2. 示例过时
**风险**: 随着 Agent 功能演进，示例可能不再准确反映触发逻辑。

**缓解策略**:
- 将示例验证纳入测试流程
- 定期审查和更新示例
- 在示例中包含版本信息

#### 3. 过度依赖示例
**风险**: Claude 可能过于字面地匹配示例，错过变体表达。

**缓解策略**:
- 包含多样化的措辞示例
- 在 commentary 中解释意图而非仅匹配关键词
- 测试各种相近的表达方式

### 边界条件

#### 数量边界
- **过少（< 2）**: 无法覆盖主要场景
- **适中（3-4）**: 最佳平衡点
- **过多（> 6）**: 描述过长，可能降低匹配精度

#### 复杂度边界
- **简单 Agent**: 2 个示例足够（显式 + 主动）
- **复杂 Agent**: 需要 4-6 个示例覆盖各种场景
- **领域特定 Agent**: 需要包含领域特定的触发情境

### 改进建议

#### 1. 负面示例支持
**建议**: 添加 `<negative-example>` 块，明确说明何时不应触发。

**示例**:
```markdown
<negative-example>
Context: User asks about general programming question
user: "What is a closure in JavaScript?"
<commentary>
This is a general question, not a code review request. Do NOT trigger code-reviewer.
</commentary>
</negative-example>
```

#### 2. 触发优先级标注
**建议**: 允许在示例中标注优先级：
```markdown
<example priority="high">
...
</example>
```

#### 3. 条件触发语法
**建议**: 支持条件表达式：
```markdown
<example condition="file_extension == '.py'">
...
</example>
```

#### 4. 自动化测试生成
**建议**: 从示例自动生成测试用例：
```bash
# 生成测试脚本
./generate-trigger-tests.sh agents/my-agent.md
# 输出: test_cases/my-agent.trigger-tests.json
```

#### 5. 触发分析工具
**建议**: 创建工具分析触发效果：
```bash
# 分析触发日志
./analyze-trigger-performance.sh --agent my-agent --log trigger.log
# 输出: 触发成功率、误触发率、未触发率
```

#### 6. 跨 Agent 协调
**建议**: 添加机制避免 Agent 冲突：
```markdown
---
name: my-agent
triggers_before: [other-agent]  # 优先于 other-agent
triggers_after: [another-agent]  # 次于 another-agent
mutually_exclusive_with: [conflicting-agent]
---
```

### 使用最佳实践

1. **具体优先**: 使用具体的用户消息而非通用表达
2. **情境完整**: 始终提供 Context 帮助理解背景
3. **注释清晰**: Commentary 应解释"为什么"而不仅是"是什么"
4. **多样化措辞**: 为相同意图提供 2-3 种不同表达方式
5. **主动 + 被动**: 同时包含显式请求和主动触发的示例
6. **边界覆盖**: 包含典型场景和边界情况的示例
7. **测试验证**: 使用示例中的措辞实际测试触发效果
8. **迭代优化**: 根据实际使用情况调整示例
