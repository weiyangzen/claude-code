# Skill-Reviewer Agent 深度研究文档

## 1. 场景与职责

### 1.1 定位与目标场景

`skill-reviewer` 是 `plugin-dev` 插件套件中的**技能质量专家**角色，专注于评估和改进 Claude Code 插件中 Skill 组件的质量。其核心价值在于：

- **Skill 创建后评审**：用户创建新 Skill 后，确保其遵循最佳实践
- **描述优化**：改进 Skill 的触发描述，提高自动加载的准确性
- **内容组织审查**：验证渐进式披露原则的正确实施
- **质量持续改进**：通过迭代反馈提升 Skill 的实用性

### 1.2 核心职责

| 职责领域 | 具体任务 |
|---------|---------|
| 结构审查 | 验证 YAML frontmatter 格式、必填字段 |
| 描述质量评估 | 检查触发短语、第三人称使用、具体性 |
| 内容质量评估 | 评估字数、写作风格、组织逻辑 |
| 渐进式披露检查 | 验证 SKILL.md 精简度、references/ 使用 |
| 支持文件审查 | 检查 references/、examples/、scripts/ 质量 |
| 改进建议 | 提供具体的优化建议和前后对比示例 |

### 1.3 触发条件

根据 agent 描述中的 `<example>` 块，触发场景包括：

1. **Skill 创建后**：用户说 "I've created a PDF processing skill"
2. **显式评审请求**：用户说 "Review my skill and tell me how to improve it"
3. **描述修改后**：用户说 "I updated the skill description, does it look good?"
4. **工作流集成**：在 `create-plugin` 命令 Phase 6 中被显式调用

---

## 2. 功能点目的

### 2.1 八步评审流程

```
┌─────────────────────────────────────────────────────────────┐
│  Step 1: Locate and Read Skill                              │
│  └── 查找 SKILL.md，读取 frontmatter 和正文                 │
├─────────────────────────────────────────────────────────────┤
│  Step 2: Validate Structure                                 │
│  └── YAML 格式、必填字段 (name, description)、正文存在性    │
├─────────────────────────────────────────────────────────────┤
│  Step 3: Evaluate Description (最关键)                      │
│  └── 触发短语、第三人称、具体性、长度、示例触发词           │
├─────────────────────────────────────────────────────────────┤
│  Step 4: Assess Content Quality                             │
│  └── 字数 (1,000-3,000)、写作风格 (祈使/不定式)、组织       │
├─────────────────────────────────────────────────────────────┤
│  Step 5: Check Progressive Disclosure                       │
│  └── SKILL.md 精简度、references/ 使用、指针清晰度          │
├─────────────────────────────────────────────────────────────┤
│  Step 6: Review Supporting Files                            │
│  └── references/、examples/、scripts/ 质量检查              │
├─────────────────────────────────────────────────────────────┤
│  Step 7: Identify Issues                                    │
│  └── 按严重度分类 (critical/major/minor)，识别反模式        │
├─────────────────────────────────────────────────────────────┤
│  Step 8: Generate Recommendations                           │
│  └── 具体修复建议、前后对比示例、按影响优先级排序           │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 各评审步骤的目的详解

#### Step 3: Description 评估 (最关键)

| 检查维度 | 目的 | 标准 |
|---------|------|------|
| 触发短语 | 确保 Skill 能被正确触发 | 包含用户实际会说的具体短语 |
| 第三人称 | 符合 Claude Code 规范 | "This skill should be used when..." |
| 具体性 | 避免模糊描述 | 具体场景，非泛泛而谈 |
| 长度 | 平衡信息量和简洁性 | 50-500 字符 |
| 示例触发词 | 列出具体查询 | "create X", "configure Y" |

**好的描述示例：**
```yaml
description: This skill should be used when the user asks to "create a hook", "add a PreToolUse hook", "validate tool use", "implement prompt-based hooks", "${CLAUDE_PLUGIN_ROOT}", "block dangerous commands"
```

**差的描述示例：**
```yaml
description: Use this skill when working with hooks.  # 错误人称，模糊
description: Load when user needs hook help.  # 非第三人称
description: Provides hook guidance.  # 无触发短语
```

#### Step 4: Content Quality 评估

| 检查维度 | 标准 | 目的 |
|---------|------|------|
| 字数 | 1,000-3,000 词 (精简专注) | 控制上下文大小 |
| 写作风格 | 祈使/不定式 ("To do X, do Y") | 客观指导，非第二人称 |
| 组织 | 清晰的章节，逻辑流程 | 易于理解和遵循 |
| 具体性 | 具体指导，非模糊建议 | 可操作性强 |

**正确的写作风格：**
```markdown
To create a hook, define the event type.
Configure the MCP server with authentication.
Validate settings before use.
```

**错误的写作风格：**
```markdown
You should create a hook by defining the event type.
You need to configure the MCP server.
You can validate settings before use.
```

#### Step 5: Progressive Disclosure 检查

渐进式披露三层结构：

```
┌─────────────────────────────────────────────────────────────┐
│  Level 1: Metadata (name + description)                     │
│  ├── 始终加载 (~100 词)                                     │
│  └── 决定何时触发 Skill                                     │
├─────────────────────────────────────────────────────────────┤
│  Level 2: SKILL.md Body                                     │
│  ├── 触发时加载 (<5k 词，推荐 1,500-2,000)                  │
│  └── 核心概念和基本流程                                     │
├─────────────────────────────────────────────────────────────┤
│  Level 3: Bundled Resources                                 │
│  ├── 需要时加载 (无限制)                                    │
│  │   ├── references/ - 详细文档                            │
│  │   ├── examples/ - 工作示例                              │
│  │   └── scripts/ - 可执行脚本                             │
│  └── 深度知识和工具                                         │
└─────────────────────────────────────────────────────────────┘
```

### 2.3 输出报告格式

评审报告采用标准化结构：

```markdown
## Skill Review: [skill-name]

### Summary
[Overall assessment and word counts]

### Description Analysis
**Current:** [Show current description]

**Issues:**
- [Issue 1 with description]
- [Issue 2...]

**Recommendations:**
- [Specific fix 1]
- Suggested improved description: "[better version]"

### Content Quality

**SKILL.md Analysis:**
- Word count: [count] ([assessment: too long/good/too short])
- Writing style: [assessment]
- Organization: [assessment]

**Issues:**
- [Content issue 1]
- [Content issue 2]

**Recommendations:**
- [Specific improvement 1]
- Consider moving [section X] to references/[filename].md

### Progressive Disclosure

**Current Structure:**
- SKILL.md: [word count]
- references/: [count] files, [total words]
- examples/: [count] files
- scripts/: [count] files

**Assessment:**
[Is progressive disclosure effective?]

**Recommendations:**
[Suggestions for better organization]

### Specific Issues

#### Critical ([count])
- [File/location]: [Issue] - [Fix]

#### Major ([count])
- [File/location]: [Issue] - [Recommendation]

#### Minor ([count])
- [File/location]: [Issue] - [Suggestion]

### Positive Aspects
- [What's done well 1]
- [What's done well 2]

### Overall Rating
[Pass/Needs Improvement/Needs Major Revision]

### Priority Recommendations
1. [Highest priority fix]
2. [Second priority]
3. [Third priority]
```

---

## 3. 具体技术实现

### 3.1 Agent 文件结构

```yaml
---
name: skill-reviewer
description: Use this agent when...  # 包含 3 个 <example> 块
model: inherit
color: cyan
tools: ["Read", "Grep", "Glob"]
---

[系统提示词 - 包含评审流程、质量标准、输出格式]
```

### 3.2 关键评审逻辑

#### 3.2.1 描述质量检查清单

```markdown
✅ 必须:
- 第三人称 ("This skill should be used when...")
- 具体触发短语 (用户实际会说的词)
- 长度适中 (50-500 字符)

❌ 避免:
- 第二人称 ("Use this skill when you...")
- 模糊描述 ("Provides guidance...")
- 无触发短语
```

#### 3.2.2 内容质量检查清单

```markdown
✅ 必须:
- 字数 1,000-3,000 词
- 祈使/不定式写作风格
- 清晰的章节组织
- 具体可操作的建议

❌ 避免:
- 字数 >5,000 词 (应移到 references/)
- 第二人称写作 ("You should...")
- 模糊建议
- 信息重复
```

#### 3.2.3 渐进式披露检查清单

```markdown
✅ 必须:
- SKILL.md 精简 (核心概念)
- 详细内容在 references/
- 工作示例在 examples/
- 工具脚本在 scripts/
- SKILL.md 明确引用这些资源

❌ 避免:
- 所有内容在 SKILL.md (>3,000 词无 references/)
- 资源存在但 SKILL.md 未引用
- 重复信息跨文件
```

### 3.3 工具使用策略

| 工具 | 用途 |
|------|------|
| Read | 读取 SKILL.md、references/、examples/、scripts/ |
| Grep | 搜索特定模式 (如检查第三人称、触发短语) |
| Glob | 发现支持目录中的文件 (references/**/*.md) |

### 3.4 反模式识别

| 反模式 | 描述 | 修复建议 |
|--------|------|---------|
| 模糊触发描述 | 无具体触发短语 | 添加 "create X", "configure Y" 等具体短语 |
| SKILL.md 过载 | >3,000 词无 references/ | 将详细内容移到 references/ |
| 第二人称写作 | "You should..." | 改为祈使式 "Do X..." |
| 缺失资源引用 | 资源存在但 SKILL.md 未提及 | 添加 "Additional Resources" 章节 |
| 重复信息 | 跨文件重复相同内容 | 合并或删除重复 |

---

## 4. 关键代码路径与文件引用

### 4.1 本 Agent 文件

```
plugins/plugin-dev/agents/skill-reviewer.md (184 行)
├── Frontmatter (lines 1-36)
│   ├── name: skill-reviewer
│   ├── description: 触发条件 + 3 个 <example> 块
│   ├── model: inherit
│   ├── color: cyan
│   └── tools: ["Read", "Grep", "Glob"]
│
└── System Prompt (lines 38-182)
    ├── Core Responsibilities (lines 40-45)
    ├── Skill Review Process 8 Steps (lines 47-97)
    ├── Quality Standards (lines 99-105)
    ├── Output Format (lines 107-174)
    └── Edge Cases (lines 176-181)
```

### 4.2 依赖的 Skill 文档

| Skill | 路径 | 用途 |
|-------|------|------|
| skill-development | `skills/skill-development/SKILL.md` | Skill 创建标准、最佳实践 |

### 4.3 skill-development SKILL.md 关键内容

位置: `plugins/plugin-dev/skills/skill-development/SKILL.md` (637 行)

核心章节：
- **Anatomy of a Skill** (lines 25-40): Skill 结构说明
- **Progressive Disclosure** (lines 77-85): 三层披露系统
- **Skill Creation Process** (lines 87-247): 6 步创建流程
- **Writing Style Requirements** (lines 362-413): 写作风格要求
- **Validation Checklist** (lines 415-449): 验证清单
- **Common Mistakes** (lines 451-539): 常见错误示例

### 4.4 调用方

| 调用方 | 位置 | 调用方式 |
|--------|------|---------|
| create-plugin 命令 | `commands/create-plugin.md` | Phase 6 显式调用 "Review with skill-reviewer" |
| skill-development skill | `skills/skill-development/SKILL.md` | Step 5 推荐使用 "Use the skill-reviewer agent" |
| 用户直接请求 | N/A | 用户说 "review my skill" 时触发 |

---

## 5. 依赖与外部交互

### 5.1 内部依赖图

```
┌─────────────────────────────────────────────────────────────────┐
│                    skill-reviewer agent                         │
│                     (agents/skill-reviewer.md)                  │
└────────────────────┬────────────────────────────────────────────┘
                     │
                     ▼
        ┌────────────────────────────┐
        │   skill-development skill   │
        │   (skills/skill-development/│
        │            SKILL.md)        │
        └────────────────────────────┘
                     │
        ┌────────────┼────────────┐
        │            │            │
        ▼            ▼            ▼
   ┌─────────┐ ┌─────────┐ ┌─────────┐
   │references│ │examples │ │ scripts │
   │   /      │ │   /     │ │   /     │
   └─────────┘ └─────────┘ └─────────┘
```

### 5.2 与 skill-development skill 的协作

在 `skills/skill-development/SKILL.md` 中明确推荐：

```markdown
### Step 5: Validate and Test

**Use the skill-reviewer agent:**
```
Ask: "Review my skill and check if it follows best practices"
```

The skill-reviewer agent will check description quality, content organization, and progressive disclosure.
```

这种设计形成了**创建-评审**的闭环：
1. `skill-development` 指导用户创建 Skill
2. `skill-reviewer` 评审创建的 Skill 质量
3. 用户根据反馈迭代改进

### 5.3 与 create-plugin 命令的集成

在 `commands/create-plugin.md` 中：

```markdown
## Phase 6: Validation & Quality Check

3. **Review with skill-reviewer** (if plugin has skills):
   - For each skill, use skill-reviewer agent
   - Check description quality, progressive disclosure, writing style
   - Apply recommendations
```

---

## 6. 风险、边界与改进建议

### 6.1 已知边界情况

| 边界情况 | 处理方式 |
|---------|---------|
| 无描述问题 | 聚焦内容和组织评审 |
| 超长 Skill (>5,000 词) | 强烈建议拆分到 references/ |
| 新 Skill (内容最少) | 提供建设性的构建指导 |
| 完美 Skill | 仅建议小的增强 |
| 引用的文件缺失 | 清晰报告错误和路径 |

### 6.2 潜在风险

| 风险 | 影响 | 缓解措施 |
|------|------|---------|
| 主观判断 | 评审标准不一致 | 提供具体的检查清单和示例 |
| 过度批评 | 用户挫败感 | 平衡负面和正面发现 |
| 建议过于笼统 | 难以执行 | 提供具体的修复示例 |
| 忽略上下文 | 误判合理性 | 考虑 Skill 的具体用途 |

### 6.3 改进建议

#### 短期改进

1. **添加自动修复功能**
   - 不仅识别问题，还提供自动修复选项
   - 例如：自动转换第二人称为祈使式

2. **增强字数统计**
   - 当前依赖估算，建议精确统计
   - 区分正文和代码块的字数

3. **添加触发测试建议**
   - 建议用户如何测试描述是否能正确触发
   - 提供测试查询模板

#### 长期改进

1. **Skill 模板推荐**
   - 根据 Skill 类型推荐合适的模板
   - 例如：工具类 Skill、知识类 Skill、工作流类 Skill

2. **历史版本对比**
   - 跟踪 Skill 的改进历史
   - 展示改进趋势

3. **社区标准集成**
   - 从社区收集高质量 Skill 作为参考
   - 动态更新最佳实践

4. **多语言支持**
   - 支持非英语 Skill 的评审
   - 考虑文化差异对写作风格的影响

### 6.4 与其他 Agent 的协作

```
┌─────────────────────────────────────────────────────────────┐
│                    Plugin Development Workflow               │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   ┌──────────────┐    ┌──────────────┐    ┌──────────────┐ │
│   │ agent-creator │──▶│   plugin     │──▶│ skill-reviewer│ │
│   │              │    │  -validator  │    │               │ │
│   └──────────────┘    └──────────────┘    └──────────────┘ │
│          │                   │                   ▲          │
│          │                   │                   │          │
│          ▼                   ▼                   │          │
│   ┌──────────────────────────────────────────────────────┐ │
│   │              create-plugin Command                    │ │
│   │  (Phase 5)        (Phase 6)         (Phase 6)        │ │
│   └──────────────────────────────────────────────────────┘ │
│                                                             │
│   skill-reviewer ◀─────────────────────────────────────────┤
│   被 skill-development skill 推荐用于 Step 5 验证          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 7. 总结

`skill-reviewer` 是 `plugin-dev` 插件套件中的专业质量评审组件，专注于提升 Skill 组件的触发准确性和内容质量。其核心优势在于：

1. **专注性**：专门针对 Skill 组件的质量维度进行深度评审
2. **实用性**：提供具体的修复建议和前后对比示例
3. **教育性**：通过评审过程传授最佳实践
4. **集成性**：与 `skill-development` 和 `create-plugin` 形成完整工作流

作为研究文档的读者，理解此 agent 的工作原理有助于：
- 创建高质量的 Skill 组件
- 理解渐进式披露原则的实践
- 掌握 Skill 描述的最佳写法
- 建立 Skill 质量评审的思维框架

与 `plugin-validator` 的全局验证不同，`skill-reviewer` 专注于 Skill 这一特定组件的深度质量提升，两者形成互补关系，共同保障插件的整体质量。
