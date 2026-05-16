# Plugin-Validator Agent 深度研究文档

## 1. 场景与职责

### 1.1 定位与目标场景

`plugin-validator` 是 `plugin-dev` 插件套件中的**质量守门员**角色，专门负责在插件开发生命周期中执行全面的结构验证。其核心价值在于：

- **开发阶段验证**：用户创建或修改插件组件后，主动触发验证以尽早发现问题
- **发布前检查**：在插件发布到市场前执行全面的合规性检查
- **持续质量保障**：作为 `/plugin-dev:create-plugin` 工作流 Phase 6 的核心验证环节

### 1.2 核心职责

| 职责领域 | 具体任务 |
|---------|---------|
| 结构验证 | 验证插件目录结构、文件组织、命名规范 |
| 清单验证 | 检查 `plugin.json` 语法、必填字段、版本格式 |
| 组件验证 | 验证 commands、agents、skills、hooks 的格式与内容 |
| MCP 验证 | 检查 `.mcp.json` 配置和服务器设置 |
| 安全检查 | 检测硬编码凭证、不安全协议、潜在安全问题 |
| 质量报告 | 生成结构化的验证报告，包含严重/警告/建议分级 |

### 1.3 触发条件

根据 agent 描述中的 `<example>` 块，触发场景包括：

1. **主动验证请求**：用户明确说 "validate my plugin", "check plugin structure"
2. **组件创建后**：用户创建新插件或添加组件后，Claude 主动建议验证
3. **清单修改后**：用户更新 `plugin.json` 后触发验证
4. **工作流集成**：在 `create-plugin` 命令的 Phase 6 中被显式调用

---

## 2. 功能点目的

### 2.1 十步验证流程

Agent 定义了系统化的 10 步验证流程：

```
┌─────────────────────────────────────────────────────────────┐
│  Step 1: Locate Plugin Root                                 │
│  └── 检查 .claude-plugin/plugin.json 存在性                 │
├─────────────────────────────────────────────────────────────┤
│  Step 2: Validate Manifest                                  │
│  └── JSON 语法、name 格式、版本语义化、作者信息             │
├─────────────────────────────────────────────────────────────┤
│  Step 3: Validate Directory Structure                       │
│  └── 标准目录检查 (commands/, agents/, skills/, hooks/)     │
├─────────────────────────────────────────────────────────────┤
│  Step 4: Validate Commands                                  │
│  └── YAML frontmatter、description、argument-hint           │
├─────────────────────────────────────────────────────────────┤
│  Step 5: Validate Agents                                    │
│  └── 调用 validate-agent.sh 或手动检查 frontmatter          │
├─────────────────────────────────────────────────────────────┤
│  Step 6: Validate Skills                                    │
│  └── SKILL.md 存在性、YAML frontmatter、子目录结构          │
├─────────────────────────────────────────────────────────────┤
│  Step 7: Validate Hooks                                     │
│  └── 调用 validate-hook-schema.sh 检查 hooks.json           │
├─────────────────────────────────────────────────────────────┤
│  Step 8: Validate MCP Configuration                         │
│  └── .mcp.json 语法、服务器配置、${CLAUDE_PLUGIN_ROOT} 使用 │
├─────────────────────────────────────────────────────────────┤
│  Step 9: Check File Organization                            │
│  └── README.md、.gitignore、LICENSE 存在性                  │
├─────────────────────────────────────────────────────────────┤
│  Step 10: Security Checks                                   │
│  └── 硬编码凭证、HTTPS/WSS 协议、安全问题检测               │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 各验证步骤的目的详解

#### Step 2: Manifest 验证

| 检查项 | 目的 | 规则 |
|-------|------|------|
| JSON 语法 | 确保文件可解析 | 使用 `jq` 或手动解析 |
| name 字段 | 插件标识符 | kebab-case, 无空格 |
| version | 版本管理 | 语义化版本 X.Y.Z |
| description | 插件描述 | 非空字符串 |
| author | 作者信息 | 有效结构 |
| mcpServers | MCP 配置 | 有效的服务器配置 |
| 未知字段 | 向前兼容 | 警告但不失败 |

#### Step 4-7: 组件验证

各组件验证的依赖工具：

| 组件 | 验证方式 | 依赖脚本 |
|------|---------|---------|
| Commands | 手动检查 frontmatter | 无 |
| Agents | validate-agent.sh | `skills/agent-development/scripts/validate-agent.sh` |
| Skills | 手动检查 SKILL.md | 无 |
| Hooks | validate-hook-schema.sh | `skills/hook-development/scripts/validate-hook-schema.sh` |

### 2.3 输出报告格式

验证报告采用标准化结构：

```markdown
## Plugin Validation Report

### Plugin: [name]
Location: [path]

### Summary
[Overall assessment - pass/fail with key stats]

### Critical Issues ([count])
- `file/path` - [Issue] - [Fix]

### Warnings ([count])
- `file/path` - [Issue] - [Recommendation]

### Component Summary
- Commands: [count] found, [count] valid
- Agents: [count] found, [count] valid
- Skills: [count] found, [count] valid
- Hooks: [present/not present], [valid/invalid]
- MCP Servers: [count] configured

### Positive Findings
- [What's done well]

### Recommendations
1. [Priority recommendation]
2. [Additional recommendation]

### Overall Assessment
[PASS/FAIL] - [Reasoning]
```

---

## 3. 具体技术实现

### 3.1 Agent 文件结构

```yaml
---
name: plugin-validator
description: Use this agent when...  # 包含 3 个 <example> 块
model: inherit
color: yellow
tools: ["Read", "Grep", "Glob", "Bash"]
---

[系统提示词 - 包含验证流程、质量标准、输出格式]
```

### 3.2 关键验证逻辑

#### 3.2.1 Agent 名称验证规则

```
格式: lowercase, numbers, hyphens only
长度: 3-50 字符
模式: 必须以字母数字开头和结尾

✅ 有效: code-reviewer, test-generator, api-docs-writer
❌ 无效: helper (太泛), -agent- (连字符开头), my_agent (下划线)
```

#### 3.2.2 描述字段验证规则

```
长度: 10-5,000 字符 (推荐 200-1,000)
必须包含: 触发条件和 <example> 块
推荐模式: "Use this agent when..."
```

#### 3.2.3 系统提示词验证规则

```
长度: 20-10,000 字符 (推荐 500-3,000)
结构: 清晰的责任、流程、输出格式
人称: 第二人称 ("You are...", "You will...")
```

### 3.3 依赖的验证脚本

#### validate-agent.sh (217 行)

位置: `plugins/plugin-dev/skills/agent-development/scripts/validate-agent.sh`

核心检查项：
- 文件存在性
- YAML frontmatter 结构 (--- 开始和结束)
- 必填字段: name, description, model, color
- name 格式验证 (长度、字符、泛型检测)
- description 长度和示例块检查
- model 有效性 (inherit/sonnet/opus/haiku)
- color 有效性 (blue/cyan/green/yellow/magenta/red)
- 系统提示词长度和第二人称检查

#### validate-hook-schema.sh (159 行)

位置: `plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh`

核心检查项：
- JSON 语法有效性
- 事件类型有效性 (PreToolUse, PostToolUse, Stop 等 9 种)
- 每个 hook 的 matcher 和 hooks 数组存在性
- hook 类型有效性 (command/prompt)
- command hook 的 command 字段存在性
- prompt hook 的 prompt 字段存在性
- 硬编码路径检测 (建议使用 ${CLAUDE_PLUGIN_ROOT})
- timeout 范围检查 (5-600 秒)

### 3.4 工具使用策略

| 工具 | 用途 |
|------|------|
| Read | 读取 plugin.json、SKILL.md、agent 文件内容 |
| Grep | 搜索特定模式 (如检查 example 块) |
| Glob | 发现组件目录中的文件 (commands/**/*.md) |
| Bash | 执行 jq 验证 JSON、运行验证脚本 |

---

## 4. 关键代码路径与文件引用

### 4.1 本 Agent 文件

```
plugins/plugin-dev/agents/plugin-validator.md (184 行)
├── Frontmatter (lines 1-37)
│   ├── name: plugin-validator
│   ├── description: 触发条件 + 3 个 <example> 块
│   ├── model: inherit
│   ├── color: yellow
│   └── tools: ["Read", "Grep", "Glob", "Bash"]
│
└── System Prompt (lines 39-182)
    ├── Core Responsibilities (lines 41-47)
    ├── Validation Process 10 Steps (lines 49-134)
    ├── Quality Standards (lines 136-141)
    ├── Output Format (lines 143-173)
    └── Edge Cases (lines 175-181)
```

### 4.2 依赖的 Skill 文档

| Skill | 路径 | 用途 |
|-------|------|------|
| agent-development | `skills/agent-development/SKILL.md` | Agent 结构参考、validate-agent.sh |
| skill-development | `skills/skill-development/SKILL.md` | Skill 结构验证参考 |
| hook-development | `skills/hook-development/SKILL.md` | Hook 验证、validate-hook-schema.sh |
| plugin-structure | `skills/plugin-structure/SKILL.md` | 目录结构标准 |
| mcp-integration | `skills/mcp-integration/SKILL.md` | MCP 配置验证 |

### 4.3 依赖的验证脚本

```
plugins/plugin-dev/
├── skills/agent-development/scripts/
│   └── validate-agent.sh          # Agent 文件验证
│
└── skills/hook-development/scripts/
    └── validate-hook-schema.sh    # Hook JSON 验证
```

### 4.4 调用方

| 调用方 | 位置 | 调用方式 |
|--------|------|---------|
| create-plugin 命令 | `commands/create-plugin.md` | Phase 6 显式调用 "Run plugin-validator agent" |
| 用户直接请求 | N/A | 用户说 "validate my plugin" 时触发 |

---

## 5. 依赖与外部交互

### 5.1 内部依赖图

```
┌─────────────────────────────────────────────────────────────────┐
│                    plugin-validator agent                       │
│                     (agents/plugin-validator.md)                │
└────────────────────┬────────────────────────────────────────────┘
                     │
        ┌────────────┼────────────┐
        │            │            │
        ▼            ▼            ▼
┌──────────────┐ ┌──────────┐ ┌──────────────┐
│   validate   │ │  skill   │ │   plugin     │
│   -agent.sh  │ │development│ │  -structure  │
└──────────────┘ └──────────┘ └──────────────┘
        │
        ▼
┌──────────────┐
│validate-hook │
│-schema.sh    │
└──────────────┘
```

### 5.2 外部工具依赖

| 工具 | 用途 | 必需性 |
|------|------|--------|
| jq | JSON 语法验证 | 强烈推荐 |
| sed/awk/grep | 文本处理 | 必需 |
| bash | 脚本执行 | 必需 |

### 5.3 与 create-plugin 命令的集成

在 `commands/create-plugin.md` 中：

```markdown
## Phase 6: Validation & Quality Check

1. **Run plugin-validator agent**:
   - Use plugin-validator agent to comprehensively validate plugin
   - Check: manifest, structure, naming, components, security
   - Review validation report

2. **Fix critical issues**:
   - Address any critical errors from validation
   - Fix any warnings that indicate real problems

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
| 最小化插件 (仅 plugin.json) | 如果清单正确则视为有效 |
| 空目录 | 警告但不失败 |
| 清单中的未知字段 | 警告但不失败 |
| 多个验证错误 | 按文件分组，优先处理严重错误 |
| 插件未找到 | 清晰的错误消息 + 指导 |
| 文件损坏 | 跳过并报告，继续验证其他文件 |

### 6.2 潜在风险

| 风险 | 影响 | 缓解措施 |
|------|------|---------|
| 验证脚本不存在 | 验证失败 | Agent 会回退到手动检查 |
| jq 未安装 | JSON 验证受限 | 使用 Read + 手动解析作为备选 |
| 大型插件性能 | 验证时间过长 | Glob 模式限制，分批处理 |
| 误报/漏报 | 质量问题 | 多层级检查 (错误/警告/建议) |

### 6.3 改进建议

#### 短期改进

1. **增强验证脚本路径检测**
   - 当前假设脚本位于固定路径
   - 建议添加路径存在性检查，提供更友好的错误消息

2. **添加并行验证**
   - 对于大型插件，可以并行验证不同组件类型
   - 使用 Task 工具并发执行

3. **缓存验证结果**
   - 对于未修改的文件，跳过重复验证
   - 使用文件哈希或修改时间

#### 长期改进

1. **自定义规则支持**
   - 允许插件定义自定义验证规则
   - 通过 `.claude-plugin/validation-rules.json`

2. **自动修复建议**
   - 不仅报告问题，还提供自动修复选项
   - 用户确认后自动应用修复

3. **验证规则版本化**
   - 随着插件规范演进，支持不同版本的验证规则
   - 在 plugin.json 中声明遵循的规范版本

4. **集成测试验证**
   - 验证 hooks 的实际执行效果
   - 模拟事件触发，验证 hook 响应

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
│          │                   │                   │          │
│          │                   │                   │          │
│          ▼                   ▼                   ▼          │
│   ┌──────────────────────────────────────────────────────┐ │
│   │              create-plugin Command                    │ │
│   │  (Phase 5)        (Phase 6)         (Phase 6)        │ │
│   └──────────────────────────────────────────────────────┘ │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 7. 总结

`plugin-validator` 是 `plugin-dev` 插件套件中的关键质量保障组件，通过系统化的 10 步验证流程确保插件符合 Claude Code 的插件规范。其核心优势在于：

1. **全面性**：覆盖从清单到组件、从结构到安全的全方位验证
2. **标准化**：提供结构化的验证报告，便于理解和修复问题
3. **可扩展性**：通过调用专门的验证脚本，易于添加新的验证规则
4. **集成性**：与 `create-plugin` 工作流无缝集成，成为开发流程的标准环节

作为研究文档的读者，理解此 agent 的工作原理有助于：
- 开发符合规范的插件
- 自定义验证流程
- 扩展验证能力
- 调试验证失败的问题
