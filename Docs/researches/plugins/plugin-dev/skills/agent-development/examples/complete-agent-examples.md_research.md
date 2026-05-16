# Complete Agent Examples 研究文档

## 场景与职责

### 文件定位
`complete-agent-examples.md` 是 Claude Code 插件开发框架中 **Agent Development Skill** 的**生产级示例集合**，位于 `plugins/plugin-dev/skills/agent-development/examples/` 目录下。

### 核心场景
该文档提供**可直接复制使用的完整 Agent 模板**，解决以下场景需求：

1. **开发者需要快速启动 Agent 开发** - 提供生产就绪的模板，无需从零开始
2. **希望遵循经过验证的最佳实践** - 所有示例均来自实际生产环境验证
3. **需要特定类型 Agent 的参考实现** - 覆盖代码审查、测试生成、文档、安全分析四大领域
4. **学习 Agent 设计的完整结构** - 通过真实示例理解 system prompt 的编写艺术

### 文档职责
- 作为**模板库 (Template Library)**，提供 4 个可直接使用的完整 Agent 配置
- 作为**教学材料**，展示不同场景下 Agent 的设计差异
- 作为**定制指南**，提供针对特定领域的调整建议
- 作为**最佳实践参考**，展示颜色、工具、结构的标准用法

---

## 功能点目的

### 1. 四大生产级 Agent 模板

| Agent 类型 | 文件命名 | 核心用途 | 颜色 | 工具集 |
|------------|----------|----------|------|--------|
| Code Reviewer | `code-reviewer.md` | 代码质量审查、安全检查、最佳实践验证 | blue | Read, Grep, Glob |
| Test Generator | `test-generator.md` | 单元测试生成、测试覆盖提升 | green | Read, Write, Grep, Bash |
| Docs Generator | `docs-generator.md` | API 文档生成、代码文档化 | cyan | Read, Write, Grep, Glob |
| Security Analyzer | `security-analyzer.md` | 安全漏洞分析、OWASP 检查 | red | Read, Grep, Glob |

### 2. 模板结构设计

每个 Agent 模板包含**标准化章节**：

```markdown
---
name: {identifier}
description: {触发条件 + 2-4个<example>块}
model: inherit
color: {语义化颜色}
tools: [{最小权限工具集}]
---

You are {专家角色}...

**Your Core Responsibilities:**
1. {主要职责}
2. {次要职责}
...

**{任务名称} Process:**
1. {步骤1}
2. {步骤2}
...

**Quality Standards:**
- {质量标准1}
- {质量标准2}
...

**Output Format:**
{结构化输出模板}

**Edge Cases:**
- {边界情况1}: {处理方式}
- {边界情况2}: {处理方式}
```

### 3. 定制化指南

文档提供三类定制维度：

#### 领域适配 (Adapt to Your Domain)
- 修改专业领域描述（如 "Python 专家" vs "React 专家"）
- 调整流程步骤以匹配特定工作流
- 添加领域特定的检查项

#### 工具权限调整 (Adjust Tool Access)
| Agent 类型 | 推荐工具集 | 使用场景 |
|------------|-----------|----------|
| 只读分析 | `["Read", "Grep", "Glob"]` | 审查、分析类 Agent |
| 生成型 | `["Read", "Write", "Grep"]` | 代码/文档生成 Agent |
| 执行型 | `["Read", "Write", "Bash", "Grep"]` | 需要运行命令的 Agent |
| 全功能 | 省略 tools 字段 | 复杂多步骤 Agent |

#### 颜色语义系统 (Customize Colors)
| 颜色 | 语义 | 适用 Agent 类型 |
|------|------|----------------|
| blue | 分析、审查、调查 | code-reviewer, analyzer |
| cyan | 文档、信息展示 | docs-generator |
| green | 生成、创建、成功导向 | test-generator |
| yellow | 验证、警告、谨慎 | validator |
| red | 安全、关键分析、错误 | security-analyzer |
| magenta | 重构、转换、创意 | transformer, creator |

---

## 具体技术实现

### 关键流程分析

#### 流程 1：Code Reviewer Agent 工作流

```
Gather Context (Glob 查找最近修改文件)
    ↓
Read Code (使用 Read 工具检查变更)
    ↓
Analyze Quality (检查重复代码、复杂度、可读性、错误处理、日志)
    ↓
Security Analysis (扫描注入漏洞、认证授权、输入验证、硬编码密钥)
    ↓
Best Practices (遵循 CLAUDE.md 标准、命名规范、测试覆盖、文档)
    ↓
Categorize Issues (按严重度分组: critical/major/minor)
    ↓
Generate Report (按输出模板格式化)
```

**质量标准要求**：
- 每个问题包含文件路径和行号（如 `src/auth.ts:42`）
- 按严重度分类（critical/major/minor）
- 建议具体且可执行（非模糊建议）
- 必要时包含代码示例
- 平衡批评与肯定（包含正面观察）

#### 流程 2：Test Generator Agent 工作流

```
Analyze Code (读取实现文件，理解函数签名、输入/输出契约、边界条件、依赖)
    ↓
Identify Test Patterns (检查现有测试：框架、文件组织、命名约定、setup/teardown)
    ↓
Design Test Cases (设计 happy path、边界条件、错误案例、边缘案例)
    ↓
Generate Tests (创建测试文件：描述性名称、Arrange-Act-Assert 结构、清晰断言、适当 mock)
    ↓
Verify (确保测试可运行且清晰)
```

**质量标准要求**：
- 测试名称清晰描述被测行为
- 每个测试聚焦单一行为
- 测试独立（无共享状态）
- 适当使用 mock（避免过度 mock）
- 覆盖边界情况和错误
- 遵循 DAMP 原则（Descriptive And Meaningful Phrases）

#### 流程 3：Security Analyzer Agent 工作流

```
Identify Attack Surface (查找用户输入点、API、数据库查询)
    ↓
Check Common Vulnerabilities (注入、认证/授权缺陷、敏感数据暴露、配置错误、不安全的反序列化)
    ↓
Analyze Patterns (输入验证、输出编码、参数化查询、最小权限原则)
    ↓
Assess Risk (按严重度和可利用性分类)
    ↓
Provide Remediation (提供具体修复方案和代码示例)
```

**质量标准要求**：
- 每个漏洞包含 CVE/CWE 引用（如适用）
- 基于 CVSS 标准的严重度评估
- 修复方案包含代码示例
- 最小化误报率

### 数据结构详解

#### Code Reviewer 输出格式

```markdown
## Code Review Summary
[2-3 句概述变更和整体质量]

## Critical Issues (Must Fix)
- `src/file.ts:42` - [问题描述] - [为何关键] - [如何修复]

## Major Issues (Should Fix)
- `src/file.ts:15` - [问题描述] - [影响] - [建议]

## Minor Issues (Consider Fixing)
- `src/file.ts:88` - [问题描述] - [建议]

## Positive Observations
- [良好实践 1]
- [良好实践 2]

## Overall Assessment
[最终结论和建议]
```

#### Security Analyzer 输出格式

```markdown
## Security Analysis Report

### Summary
[高级安全态势评估]

### Critical Vulnerabilities ([count])
- **[漏洞类型]** at `file:line`
  - Risk: [安全影响描述]
  - How to Exploit: [攻击场景]
  - Fix: [具体修复方案及代码示例]

### Medium/Low Vulnerabilities
[...]

### Security Best Practices Recommendations
[...]

### Overall Risk Assessment
[High/Medium/Low 及理由]
```

### 边界情况处理矩阵

| Agent 类型 | 边界情况 | 处理策略 |
|------------|----------|----------|
| Code Reviewer | 无问题发现 | 提供正面验证，说明检查了哪些内容 |
| Code Reviewer | 问题过多 (>20) | 按类型分组，优先显示前 10 个 critical/major |
| Code Reviewer | 代码意图不明确 | 注明歧义，请求澄清 |
| Code Reviewer | 缺少上下文 (无 CLAUDE.md) | 应用通用最佳实践 |
| Code Reviewer | 大量变更 | 优先关注最有影响力的文件 |
| Test Generator | 无现有测试 | 遵循最佳实践创建新测试文件 |
| Test Generator | 已有测试文件 | 保持风格一致性添加新测试 |
| Test Generator | 行为不明确 | 为可观察行为添加测试，注明不确定性 |
| Test Generator | 复杂 mock | 优先集成测试或最小化 mock |
| Test Generator | 无法测试的代码 | 建议重构以提高可测试性 |
| Docs Generator | 私有/内部代码 | 仅在请求时文档化 |
| Docs Generator | 复杂 API | 分节展示，提供多个示例 |
| Docs Generator | 废弃代码 | 标记为废弃并提供迁移指南 |
| Security Analyzer | 无漏洞 | 确认安全审查完成，说明检查了哪些内容 |
| Security Analyzer | 误报 | 报告前验证 |
| Security Analyzer | 不确定的漏洞 | 标记为 "potential" 并注明警告 |
| Security Analyzer | 超出范围项 | 注明但不深入 |

---

## 关键代码路径与文件引用

### 直接依赖

| 文件路径 | 关系 | 说明 |
|----------|------|------|
| `SKILL.md` | 父文档 | Agent Development Skill 主入口 |
| `agent-creation-prompt.md` | 同级文档 | AI 辅助生成流程 |
| `references/system-prompt-design.md` | 参考文档 | System Prompt 设计模式 |
| `references/triggering-examples.md` | 参考文档 | `<example>` 块最佳实践 |

### 相关文件网络

```
complete-agent-examples.md (本文档)
    ├── SKILL.md                                    ← Skill 主文档
    ├── agent-creation-prompt.md                    ← AI 辅助生成流程
    ├── references/system-prompt-design.md          ← System Prompt 设计模式
    ├── references/triggering-examples.md           ← 触发示例最佳实践
    ├── references/agent-creation-system-prompt.md  ← Agent 创建系统提示
    └── agents/agent-creator.md                     ← 实际 Agent 实现
```

### 验证脚本引用

文档中提到的验证命令：
```bash
# 验证 Agent 结构
./scripts/validate-agent.sh

# 测试触发逻辑
# (通过真实场景测试)
```

---

## 依赖与外部交互

### 内部依赖

1. **Agent Development Skill** (`SKILL.md`)
   - 提供 Agent 文件格式规范
   - 定义 frontmatter 字段要求
   - 说明验证规则

2. **System Prompt 设计参考** (`references/system-prompt-design.md`)
   - 提供 4 种设计模式（分析型、生成型、验证型、编排型）
   - 定义写作风格指南
   - 提供常见陷阱和避免方法

3. **触发示例参考** (`references/triggering-examples.md`)
   - 提供 `<example>` 块的标准格式
   - 说明 4 种示例类型（显式请求、主动触发、隐式请求、工具使用模式）
   - 提供调试触发问题的指南

### 外部交互

| 交互方 | 交互方式 | 说明 |
|--------|----------|------|
| Claude Code 核心 | Agent 工具调用 | Agent 通过 Task 工具被调用 |
| 文件系统 | Read/Write/Glob/Grep | Agent 使用工具分析/生成代码 |
| 验证脚本 | Bash 执行 | 运行 validate-agent.sh 检查 |

### 与 agent-creation-prompt.md 的关系

| 对比维度 | complete-agent-examples.md | agent-creation-prompt.md |
|----------|---------------------------|-------------------------|
| **目标用户** | 需要现成模板的开发者 | 希望 AI 辅助生成的开发者 |
| **使用方式** | 复制 → 修改 → 使用 | 描述需求 → AI 生成 → 微调 |
| **内容深度** | 完整的可直接使用的 Agent | 生成流程和示例 |
| **灵活性** | 基于成熟模板定制 | 从需求直接生成 |
| **适用场景** | 已知 Agent 类型，快速启动 | 探索性创建新类型 Agent |

---

## 风险、边界与改进建议

### 潜在风险

#### 风险 1：模板僵化导致创新受限
- **症状**：开发者直接复制模板而不理解设计原理
- **影响**：产生大量结构相似但功能不匹配的 Agent
- **缓解**：
  - 强调模板是**起点**而非终点
  - 提供详细的定制指南
  - 鼓励根据实际需求调整结构

#### 风险 2：工具权限过于宽泛
- **症状**：Agent 被授予不必要的工具访问权限
- **影响**：安全风险、意外副作用
- **缓解**：
  - 严格遵循最小权限原则
  - 文档中提供工具集选择指南
  - 审查时检查 tools 字段

#### 风险 3：触发条件重叠
- **症状**：多个 Agent 在相似场景触发，导致冲突
- **影响**：Claude 选择错误的 Agent，用户体验不一致
- **缓解**：
  - 确保每个 Agent 的 description 具有独特性
  - 使用 `<commentary>` 明确区分触发场景
  - 定期测试触发逻辑

#### 风险 4：System Prompt 过长
- **症状**：Agent 行为不一致或响应缓慢
- **边界**：建议保持在 10,000 字符以内
- **缓解**：
  - 遵循长度指南（标准 Agent 1,000-2,000 词）
  - 定期审查和精简 system prompt
  - 将复杂逻辑拆分为多个 Agent

### 边界情况处理

文档中已识别的关键边界：

1. **大量输入处理**
   - Code Reviewer：>20 个问题时分组，优先显示前 10 个
   - Security Analyzer：大量漏洞时按类型分组

2. **缺失上下文**
   - 无 CLAUDE.md 时应用通用最佳实践
   - 代码意图不明确时请求澄清

3. **空/无结果场景**
   - 无问题发现时提供正面确认
   - 明确说明检查了哪些内容

4. **复杂/模糊需求**
   - 测试生成时优先测试可观察行为
   - 安全分析时标记不确定发现为 "potential"

### 改进建议

#### 短期改进

1. **添加更多 Agent 模板**
   - Performance Analyzer（性能分析）- yellow
   - Refactoring Assistant（重构辅助）- magenta
   - Dependency Checker（依赖检查）- blue
   - Migration Helper（迁移助手）- cyan

2. **提供领域特定变体**
   ```
   examples/
   ├── web-development/
   │   ├── react-component-reviewer.md
   │   └── api-endpoint-tester.md
   ├── data-science/
   │   ├── model-evaluator.md
   │   └── data-validator.md
   └── devops/
       ├── dockerfile-linter.md
       └── ci-config-validator.md
   ```

3. **增强定制指南**
   - 添加 "如何选择合适的模板" 决策树
   - 提供工具集选择的详细说明
   - 添加颜色选择的语义指南

4. **添加验证清单**
   ```markdown
   ## 使用模板前的检查清单
   
   - [ ] identifier 符合命名规范
   - [ ] description 包含 2-4 个具体示例
   - [ ] model 选择合适（复杂任务用 sonnet，简单任务用 inherit）
   - [ ] color 符合语义约定
   - [ ] tools 遵循最小权限原则
   - [ ] system prompt 包含所有标准章节
   - [ ] edge cases 覆盖了预期异常场景
   - [ ] 运行了 validate-agent.sh 验证结构
   ```

#### 长期改进

1. **交互式模板选择器**
   - 开发 CLI 工具引导用户选择合适模板
   - 根据用户输入推荐定制建议
   - 自动生成初始配置

2. **模板版本管理**
   - 为模板添加版本号
   - 提供升级指南
   - 维护变更日志

3. **社区模板市场**
   - 建立模板共享平台
   - 支持评分和评论
   - 提供模板搜索和分类

4. **自动化测试套件**
   - 为每个模板提供测试用例
   - 验证触发逻辑
   - 测试边界情况处理

### 最佳实践总结

基于本文档的 Agent 开发最佳实践：

1. **选择合适的起点**：
   - 已知 Agent 类型 → 使用 complete-agent-examples.md 模板
   - 探索新类型 → 使用 agent-creation-prompt.md AI 辅助生成

2. **严格遵循结构规范**：
   - 使用标准 system prompt 章节（职责 → 流程 → 标准 → 输出 → 边界）
   - 保持第二人称写作风格
   - 确保具体而非模糊

3. **精心设计触发条件**：
   - 包含 2-4 个不同场景的示例
   - 覆盖显式和主动触发
   - 使用 `<commentary>` 解释触发原因

4. **优化工具权限**：
   - 只授予必要的工具访问
   - 定期审查和精简
   - 遵循最小权限原则

5. **全面测试**：
   - 验证结构：运行 validate-agent.sh
   - 测试触发：使用真实场景验证
   - 测试边界：验证异常处理

6. **持续迭代**：
   - 根据实际使用情况调整 system prompt
   - 收集反馈并优化触发条件
   - 定期审查和更新
