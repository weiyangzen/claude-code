# Agent Creation Prompt 研究文档

## 场景与职责

### 文件定位
`agent-creation-prompt.md` 是 Claude Code 插件开发框架中 **Agent Development Skill** 的核心示例文档，位于 `plugins/plugin-dev/skills/agent-development/examples/` 目录下。

### 核心场景
该文档提供了一个**AI 辅助生成 Agent 的完整模板**，解决以下场景需求：

1. **开发者想要创建自定义 Agent 但不知从何开始** - 提供结构化的生成流程
2. **需要确保 Agent 配置符合 Claude Code 最佳实践** - 提供经过验证的模板和示例
3. **希望快速生成高质量的 Agent 配置** - 通过与 Claude 对话直接生成 JSON 配置

### 文档职责
- 作为**用户-facing 的模板指南**，教导开发者如何使用 AI 辅助方式创建 Agent
- 提供**完整的四步工作流**：描述需求 → 使用生成提示 → 获取 JSON → 转换为 Agent 文件
- 包含**三个完整的真实示例**：代码审查 Agent、测试生成 Agent、文档生成 Agent

---

## 功能点目的

### 1. 四步使用模式 (Usage Pattern)

| 步骤 | 目的 | 关键动作 |
|------|------|----------|
| Step 1 | 明确 Agent 需求 | 思考任务类型、触发时机、主动/被动模式 |
| Step 2 | 使用生成提示 | 向 Claude 发送结构化提示，加载 system prompt |
| Step 3 | 获取 JSON 配置 | Claude 返回包含 identifier/whenToUse/systemPrompt 的 JSON |
| Step 4 | 转换为 Agent 文件 | 将 JSON 转换为带 YAML frontmatter 的 Markdown 文件 |

### 2. 结构化生成提示

核心提示模板：
```
Create an agent configuration based on this request: "[YOUR DESCRIPTION]"

Return ONLY the JSON object, no other text.
```

**设计意图**：
- 强制 Claude 返回**纯 JSON** 格式，便于程序化解析
- 将自然语言需求**结构化**为机器可读的 Agent 配置
- 利用 Claude 的**system prompt 加载机制**确保输出质量

### 3. JSON 到 Agent 文件的转换映射

```json
{
  "identifier": "agent-name",           →  frontmatter: name
  "whenToUse": "Use this agent when...", →  frontmatter: description  
  "systemPrompt": "You are..."           →  markdown body
}
```

**转换规则**：
- JSON 字段 → YAML frontmatter 字段的**一一映射**
- `identifier` 必须满足：3-50 字符、小写、连字符分隔、 alphanumeric 开头结尾
- `whenToUse` 必须包含 `<example>` 块展示触发场景
- `systemPrompt` 必须使用第二人称（"You are..."）

---

## 具体技术实现

### 关键流程

#### 流程 1：AI 辅助 Agent 生成

```
用户描述需求
    ↓
Claude (加载 agent-creation-system-prompt)
    ↓
提取核心意图 → 设计专家人设 → 构建系统提示 → 优化性能 → 创建标识符 → 生成示例
    ↓
返回标准 JSON {identifier, whenToUse, systemPrompt}
    ↓
用户转换为 agents/[identifier].md 文件
    ↓
验证: ./scripts/validate-agent.sh agents/[identifier].md
```

#### 流程 2：JSON 到 Markdown 的转换

输入 JSON：
```json
{
  "identifier": "code-quality-reviewer",
  "whenToUse": "Use this agent when... Examples:\n\n<example>...",
  "systemPrompt": "You are an expert code quality reviewer..."
}
```

输出 Markdown：
```markdown
---
name: code-quality-reviewer
description: Use this agent when... Examples:

<example>
...
</example>

model: inherit
color: blue
tools: ["Read", "Grep", "Glob"]
---

You are an expert code quality reviewer...
```

### 数据结构

#### Agent 文件标准结构

```markdown
---
name: {identifier}                    # 必填: 3-50字符, 小写连字符
description: {whenToUse}              # 必填: 触发条件 + <example>块
model: inherit|sonnet|opus|haiku      # 必填: 推荐 inherit
color: blue|cyan|green|yellow|magenta|red  # 必填: 视觉标识
tools: ["Read", "Write", ...]         # 可选: 工具白名单
---

{systemPrompt}                        # Markdown body: 系统提示
```

#### System Prompt 标准章节

1. **角色定义**：`You are [role] specializing in [domain]`
2. **核心职责**：`**Your Core Responsibilities:**` 编号列表
3. **处理流程**：`**[Task] Process:**` 分步骤说明
4. **质量标准**：`**Quality Standards:**`  bullet 列表
5. **输出格式**：`**Output Format:**` 结构化模板
6. **边界情况**：`**Edge Cases:**` 异常处理说明

### 协议与约定

#### 颜色语义约定

| 颜色 | 语义 | 适用场景 |
|------|------|----------|
| blue | 分析、审查、调查 | code-reviewer, analyzer |
| cyan | 文档、信息 | docs-generator |
| green | 生成、创建、成功导向 | test-generator |
| yellow | 验证、警告、谨慎 | validator |
| red | 安全、关键分析、错误 | security-analyzer |
| magenta | 重构、转换、创意 | agent-creator |

#### 工具权限最小化原则

```yaml
# 只读分析 Agent
tools: ["Read", "Grep", "Glob"]

# 代码生成 Agent  
tools: ["Read", "Write", "Grep"]

# 测试执行 Agent
tools: ["Read", "Bash", "Grep"]

# 全功能 Agent (省略 tools 字段)
```

---

## 关键代码路径与文件引用

### 直接依赖

| 文件路径 | 关系 | 说明 |
|----------|------|------|
| `references/agent-creation-system-prompt.md` | 被引用 | 提供实际的 system prompt 内容 |
| `SKILL.md` | 父文档 | Agent Development Skill 的主入口 |
| `agents/agent-creator.md` | 实现参考 | 实际使用此模式的 Agent 实现 |

### 相关文件网络

```
agent-creation-prompt.md (本文档)
    ├── references/agent-creation-system-prompt.md  ← 核心 system prompt
    ├── references/system-prompt-design.md          ← 系统提示设计模式
    ├── references/triggering-examples.md           ← <example>块最佳实践
    ├── SKILL.md                                    ← Skill 主文档
    └── complete-agent-examples.md                  ← 完整示例集合
```

### 验证脚本引用

文档中提到的验证命令：
```bash
# 验证 Agent 结构
./scripts/validate-agent.sh agents/your-agent.md

# 检查触发是否正常工作
# (通过实际测试场景验证)
```

---

## 依赖与外部交互

### 内部依赖

1. **Agent 创建系统提示** (`references/agent-creation-system-prompt.md`)
   - 文档中 Step 2 提到的 "agent-creation-system-prompt" 即指此文件
   - 包含完整的 6 步 Agent 生成逻辑

2. **Claude Code 核心功能**
   - 依赖 Claude 的 **Agent 工具调用机制**
   - 依赖 **system prompt 加载** 功能
   - 依赖 **JSON 模式输出** 能力

### 外部交互

| 交互方 | 交互方式 | 说明 |
|--------|----------|------|
| Claude AI | API/对话 | 发送生成提示，接收 JSON 响应 |
| 文件系统 | Write 工具 | 创建 `agents/[identifier].md` 文件 |
| 验证脚本 | Bash 执行 | 运行 `validate-agent.sh` 检查结构 |

### 与 complete-agent-examples.md 的关系

| 对比维度 | agent-creation-prompt.md | complete-agent-examples.md |
|----------|-------------------------|---------------------------|
| **定位** | AI 辅助生成模板 | 手动复制模板 |
| **使用方式** | 与 Claude 对话生成 | 直接复制修改 |
| **内容** | 生成流程 + 示例 | 4 个完整可直接使用的 Agent |
| **适用场景** | 快速生成新 Agent | 基于成熟模板定制 |

---

## 风险、边界与改进建议

### 潜在风险

#### 风险 1：JSON 解析失败
- **症状**：Claude 返回的 JSON 包含额外文本，导致解析失败
- **缓解**：提示中明确要求 "Return ONLY the JSON object, no other text"
- **建议**：实现 JSON 提取器，从响应中自动提取 JSON 块

#### 风险 2：生成的 Agent 触发不准确
- **症状**：Agent 无法在正确场景触发，或误触发
- **原因**：`whenToUse` 中的示例不够具体或覆盖不全
- **缓解**：
  - 确保每个 Agent 有 2-4 个不同场景的示例
  - 包含主动触发和被动触发两种模式
  - 使用 `<commentary>` 明确解释触发原因

#### 风险 3：System Prompt 过于冗长
- **症状**：Agent 行为不一致或性能下降
- **边界**：建议 system prompt 保持在 10,000 字符以内
- **缓解**：遵循 `system-prompt-design.md` 中的长度指南

### 边界情况处理

文档中已识别的边界：

1. **模糊的用户请求**
   - 处理：先生成基础版本，再建议用户迭代优化
   - 改进：添加澄清问题环节

2. **与现有 Agent 冲突**
   - 处理：提示命名冲突，建议不同作用域或名称
   - 改进：添加自动冲突检测

3. **复杂需求需要多个 Agent**
   - 处理：建议拆分为多个专业 Agent
   - 改进：提供 Agent 拆分指导原则

4. **用户指定特定工具访问**
   - 处理：在配置中尊重用户请求
   - 改进：提供工具权限最佳实践说明

### 改进建议

#### 短期改进

1. **添加 JSON Schema 验证**
   ```json
   {
     "$schema": "http://json-schema.org/draft-07/schema#",
     "type": "object",
     "required": ["identifier", "whenToUse", "systemPrompt"],
     "properties": {
       "identifier": {
         "type": "string",
         "pattern": "^[a-z][a-z0-9-]{1,48}[a-z0-9]$"
       },
       "whenToUse": {
         "type": "string",
         "minLength": 10,
         "maxLength": 5000
       },
       "systemPrompt": {
         "type": "string",
         "minLength": 20,
         "maxLength": 10000
       }
     }
   }
   ```

2. **提供自动化转换脚本**
   - 创建 `convert-agent-json.sh` 脚本
   - 自动将 JSON 转换为标准 Markdown 格式
   - 处理 frontmatter 的 YAML 转义

3. **增强验证步骤**
   - 添加 `test-agent-trigger.sh` 引用
   - 提供触发测试的示例命令

#### 长期改进

1. **集成到 CLI 工具**
   - 开发 `claude-code create-agent` 命令
   - 交互式收集需求
   - 自动生成并保存 Agent 文件

2. **Agent 市场/模板库**
   - 建立可共享的 Agent 模板库
   - 支持模板搜索和一键安装

3. **可视化 Agent 编辑器**
   - 提供 Web UI 编辑 frontmatter
   - 实时预览 system prompt 效果
   - 内置触发测试工具

### 最佳实践总结

基于本文档的 Agent 创建最佳实践：

1. **始终使用 AI 辅助生成** 作为起点，然后手动微调
2. **确保 identifier 具有描述性**，避免通用名称如 "helper"
3. **在 description 中包含多样化的示例**，覆盖不同触发场景
4. **保持 system prompt 结构一致**：职责 → 流程 → 标准 → 输出 → 边界
5. **验证后再使用**：运行 validate-agent.sh 检查结构正确性
6. **测试触发逻辑**：用真实场景验证 Agent 能在正确时机触发
