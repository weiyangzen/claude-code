# System Prompt Design Patterns 深度研究文档

## 场景与职责

### 定位与用途

`system-prompt-design.md` 是 Claude Code 插件开发中**系统提示词设计的权威参考手册**。该文件提供了经过生产验证的系统提示词结构模板和设计模式，帮助开发者创建能够自主运行、输出高质量结果的 Agent。

### 核心职责

1. **标准化提示词结构**：提供统一的系统提示词组织框架
2. **设计模式库**：针对不同类型的 Agent（分析型、生成型、验证型、编排型）提供专用模板
3. **最佳实践指南**：涵盖语气、清晰度、可操作性等方面的写作指导
4. **反模式警示**：通过对比展示常见错误和正确做法

### 使用场景

| 场景 | 说明 |
|------|------|
| 新建 Agent | 作为系统提示词的起点模板 |
| Agent 优化 | 诊断现有 Agent 提示词问题并改进 |
| 团队协作 | 统一团队内 Agent 写作风格 |
| 培训学习 | 帮助新开发者理解高质量提示词的特征 |

---

## 功能点目的

### 1. 核心结构模板

文件定义了所有 Agent 系统提示词应遵循的基础结构：

```markdown
You are [specific role] specializing in [specific domain].

**Your Core Responsibilities:**
1. [Primary responsibility]
2. [Secondary responsibility]
...

**[Task Name] Process:**
1. [Step 1]
2. [Step 2]
...

**Quality Standards:**
- [Standard 1]
- [Standard 2]
...

**Output Format:**
Provide results structured as:
- [Component 1]
- [Component 2]
...

**Edge Cases:**
Handle these situations:
- [Edge case 1]: [Handling approach]
```

**设计意图**：
- **角色定义**：建立 Agent 的专业身份和能力边界
- **职责清单**：明确核心任务，避免范围蔓延
- **流程步骤**：提供可执行的操作序列
- **质量标准**：定义输出质量的衡量基准
- **输出格式**：确保结果的一致性和可用性
- **边界情况**：增强 Agent 的鲁棒性

### 2. 四种 Agent 类型模式

#### Pattern 1: Analysis Agents（分析型）

**适用场景**：代码审查、PR 分析、文档审查

**核心流程**：
```
Gather Context → Initial Scan → Deep Analysis → Synthesize → Prioritize → Generate Report
```

**关键特征**：
- 强调文件和行号引用
- 严重级别分类（Critical/Major/Minor）
- 正面观察平衡批评

#### Pattern 2: Generation Agents（生成型）

**适用场景**：代码生成、测试生成、文档生成

**核心流程**：
```
Understand Requirements → Gather Context → Design Structure → Generate Content → Validate → Document
```

**关键特征**：
- 遵循项目约定（CLAUDE.md）
- 包含错误处理
- 清晰命名和文档

#### Pattern 3: Validation Agents（验证型）

**适用场景**：配置验证、代码检查、合规审查

**核心流程**：
```
Load Criteria → Scan Target → Check Rules → Collect Violations → Assess Severity → Determine Result
```

**关键特征**：
- 明确的通过/失败判定
- 具体位置和修复建议
- 最小化误报

#### Pattern 4: Orchestration Agents（编排型）

**适用场景**：多步骤工作流、工具协调、复杂任务管理

**核心流程**：
```
Plan → Prepare → Execute Phases → Monitor → Verify → Report
```

**关键特征**：
- 阶段化执行
- 错误处理和重试
- 进度报告

### 3. 写作风格指南

#### 第二人称原则

```markdown
✅ You are responsible for...
✅ You will analyze...
❌ The agent is responsible for...
❌ I will analyze...
```

**目的**：建立 Agent 的主体意识，使其以第一人称执行任务。

#### 具体性原则

```markdown
✅ Check for SQL injection by examining all database queries for parameterization
❌ Look for security issues

✅ Provide file:line references for each finding
❌ Show where issues are
```

**目的**：消除歧义，确保 Agent 知道具体如何执行。

### 4. 长度指导原则

| 类型 | 字数 | 适用场景 |
|------|------|----------|
| Minimum Viable | ~500 | 简单任务，快速原型 |
| Standard | ~1,000-2,000 | 大多数生产 Agent |
| Comprehensive | ~2,000-5,000 | 复杂任务，关键系统 |
| Avoid | >10,000 | 性能下降，收益递减 |

---

## 具体技术实现

### 关键流程

#### 流程 1：提示词构建流程

```
确定 Agent 类型（分析/生成/验证/编排）
    ↓
选择对应 Pattern 模板
    ↓
填充角色和领域描述
    ↓
列出核心职责（3-8 项）
    ↓
设计分步流程（5-12 步）
    ↓
定义质量标准（3-5 条）
    ↓
指定输出格式
    ↓
列举边界情况（3-5 种）
    ↓
验证长度和完整性
```

#### 流程 2：质量检查流程

```markdown
### Test Completeness

Can the agent handle these based on system prompt alone?

- [ ] Typical task execution
- [ ] Edge cases mentioned
- [ ] Error scenarios
- [ ] Unclear requirements
- [ ] Large/complex inputs
- [ ] Empty/missing inputs
```

### 数据结构

#### 分析型 Agent 输出结构

```markdown
## Summary
[2-3 sentence overview]

## Critical Issues
- [file:line] - [Issue description] - [Recommendation]

## Major Issues
[...]

## Minor Issues
[...]

## Recommendations
[...]
```

#### 验证型 Agent 输出结构

```markdown
## Validation Result: [PASS/FAIL]

## Summary
[Overall assessment]

## Violations Found: [count]
### Critical ([count])
- [Location]: [Issue] - [Fix]

### Warnings ([count])
- [Location]: [Issue] - [Fix]

## Recommendations
[How to fix violations]
```

### 协议与约定

#### 严重级别分类协议

| 级别 | 定义 | 响应要求 |
|------|------|----------|
| Critical | 安全漏洞、功能缺陷 | 必须修复 |
| Major | 显著影响质量 | 应该修复 |
| Minor | 风格问题 | 建议修复 |

#### 文件引用协议

```markdown
✅ `src/auth.ts:42`
✅ `file.ts:15`
❌ line 42
❌ auth file
```

---

## 关键代码路径与文件引用

### 核心文件关系图

```
plugins/plugin-dev/skills/agent-development/
├── SKILL.md                              # 引用本文件作为系统提示词设计参考
├── references/
│   ├── system-prompt-design.md          # ← 本文件
│   ├── agent-creation-system-prompt.md   # 引用本文件的设计模式
│   └── triggering-examples.md            # 配套文件（触发示例）
├── examples/
│   ├── agent-creation-prompt.md          # 应用本文件的模板
│   └── complete-agent-examples.md        # 应用本文件的完整示例
└── scripts/
    └── validate-agent.sh                 # 验证系统提示词长度和结构
```

### 调用关系

#### 被调用方（消费者）

| 文件 | 引用方式 | 用途 |
|------|----------|------|
| `SKILL.md` | 链接引用 | 引导用户查阅详细设计模式 |
| `agent-creation-system-prompt.md` | 概念引用 | 基于本文件模式生成提示词 |
| `complete-agent-examples.md` | 模板应用 | 4 个完整示例均遵循本文件模式 |
| `validate-agent.sh` | 规则实现 | 验证脚本检查本文件定义的结构 |

#### 调用方（依赖）

| 文件 | 依赖关系 |
|------|----------|
| `triggering-examples.md` | 配套使用，共同构成完整 Agent 设计指南 |

### 关键代码片段

#### 1. 验证脚本中的结构检查

文件：`plugins/plugin-dev/skills/agent-development/scripts/validate-agent.sh`（第 189-202 行）

```bash
# Check for second person
if ! echo "$SYSTEM_PROMPT" | grep -q "You are\|You will\|Your"; then
  echo "⚠️  System prompt should use second person (You are..., You will...)"
  ((warning_count++))
fi

# Check for structure
if ! echo "$SYSTEM_PROMPT" | grep -qi "responsibilities\|process\|steps"; then
  echo "💡 Consider adding clear responsibilities or process steps"
fi

if ! echo "$SYSTEM_PROMPT" | grep -qi "output"; then
  echo "💡 Consider defining output format expectations"
fi
```

**说明**：验证脚本将本文件定义的最佳实践编码为自动化检查。

#### 2. 完整示例中的应用

文件：`plugins/plugin-dev/skills/agent-development/examples/complete-agent-examples.md`

以 Code Review Agent 为例：

```markdown
You are an expert code quality reviewer specializing in identifying issues...

**Your Core Responsibilities:**
1. Analyze code changes for quality issues...
2. Identify security vulnerabilities...
3. Check adherence to project best practices...
4. Provide specific, actionable feedback...
5. Recognize and commend good practices

**Code Review Process:**
1. **Gather Context**: Use Glob to find recently modified files...
2. **Read Code**: Use Read tool to examine changed files...
3. **Analyze Quality**: [...]
4. **Security Analysis**: [...]
5. **Best Practices**: [...]
6. **Categorize Issues**: [...]
7. **Generate Report**: [...]

**Quality Standards:**
- Every issue includes file path and line number...
- Issues categorized by severity with clear criteria...
- Recommendations are specific and actionable...
- Include code examples in recommendations...
- Balance criticism with recognition...

**Output Format:**
## Code Review Summary
[...]

**Edge Cases:**
- No issues found: [...]
- Too many issues (>20): [...]
- Unclear code intent: [...]
```

**说明**：完整遵循本文件定义的 Pattern 1（Analysis Agents）结构。

---

## 依赖与外部交互

### 内部依赖

| 依赖项 | 类型 | 说明 |
|--------|------|------|
| `triggering-examples.md` | 配套文件 | 共同构成完整 Agent 设计指南 |
| `SKILL.md` | 使用指南 | 指导用户如何使用本文件 |

### 外部依赖

| 依赖项 | 说明 |
|--------|------|
| Claude Code Agent 运行时 | 执行本文件定义的提示词 |
| CLAUDE.md（可选） | 项目特定上下文，影响质量标准定义 |

### 交互协议

#### 与 Agent 创建流程的集成

```
用户请求创建 Agent
    ↓
agent-creation-system-prompt.md 生成 JSON
    ↓
使用 system-prompt-design.md 作为结构指南
    ↓
生成符合 Pattern 的系统提示词
    ↓
validate-agent.sh 验证结构合规性
```

---

## 风险、边界与改进建议

### 已知风险

#### 风险 1：模板僵化

**表现**：开发者机械套用模板，忽视任务特殊性。

**缓解措施**：
- 强调模板应"customize for your domain"
- 提供多种变体示例
- 鼓励根据实际效果迭代

#### 风险 2：过度工程

**表现**：简单任务使用 Comprehensive 级别提示词，造成资源浪费。

**缓解措施**：
- 明确长度指导原则
- 提供 Minimum Viable 模板
- 验证脚本警告过长提示词

#### 风险 3：上下文窗口限制

**表现**：系统提示词 + 对话历史超出模型上下文限制。

**缓解措施**：
- 建议避免 >10,000 字符
- 使用 `inherit` 模型让 Agent 复用父级上下文策略

### 边界条件

| 边界 | 限制 | 后果 |
|------|------|------|
| 最小长度 | ~500 字符 | 指导不足，Agent 行为不稳定 |
| 最大长度 | ~10,000 字符 | 性能下降，上下文挤压 |
| 职责数量 | 3-8 项 | 过少不全面，过多难聚焦 |
| 流程步骤 | 5-12 步 | 过少不详细，过多难执行 |

### 改进建议

#### 建议 1：增加交互式提示词生成器

创建脚本根据用户输入自动选择合适的 Pattern 并生成模板：

```bash
./scripts/generate-prompt-template.sh
# 询问：Agent 类型？领域？核心任务？
# 输出：定制化模板
```

#### 建议 2：添加 A/B 测试指南

```markdown
## Testing Prompt Variants

To optimize your system prompt:

1. Create 2-3 variants with different:
   - Process step granularity
   - Output format structures
   - Edge case coverage

2. Test with identical inputs

3. Evaluate on:
   - Output quality
   - Consistency
   - Error handling
```

#### 建议 3：增加多语言支持指南

```markdown
## Multi-language Agents

When creating agents for non-English contexts:

- Write system prompt in the target language
- Maintain the same structural elements
- Adjust examples to cultural context
- Test with native speakers
```

#### 建议 4：版本化 Pattern

```markdown
---
pattern_version: 2.0
compatible_with: Claude Code >= 2.0
---
```

**理由**：随着 Claude Code 演进，Pattern 可能需要更新。

### 生产环境注意事项

1. **迭代优化**：首次创建的 Agent 很少完美，应根据实际运行结果调整提示词
2. **测试覆盖**：为每个 Agent 创建测试用例集，验证各种输入下的行为
3. **监控反馈**：收集 Agent 输出质量反馈，持续改进提示词
4. **文档同步**：提示词更新时同步更新相关文档和示例

---

## 总结

`system-prompt-design.md` 是 Claude Code Agent 生态系统的**设计规范核心**。它将模糊的艺术性提示词工程转化为可学习、可验证、可复用的系统化方法。

该文件的核心价值在于：
- **结构化思维**：将 Agent 能力分解为可管理的组件
- **模式语言**：为常见 Agent 类型提供经过验证的解决方案
- **质量基准**：定义了高质量系统提示词的衡量标准
- **学习路径**：从 Minimum Viable 到 Comprehensive 的渐进式指导

对于插件开发者而言，掌握本文件内容是创建高效、可靠 Agent 的必备技能。
