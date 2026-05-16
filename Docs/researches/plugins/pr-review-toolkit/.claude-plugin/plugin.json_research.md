# plugins/pr-review-toolkit/.claude-plugin/plugin.json 研究

## 场景与职责

`plugins/pr-review-toolkit/.claude-plugin/plugin.json` 是 `pr-review-toolkit` 插件的 manifest 入口（`plugins/pr-review-toolkit/.claude-plugin/plugin.json:1-9`）。

它在运行时不直接执行 PR 审查，而是承担“可发现、可注册、可展示”的基础职责：

1. 插件身份声明：提供 `name/version/description/author`，用于插件识别、版本与归属信息（`plugins/pr-review-toolkit/.claude-plugin/plugin.json:2-8`）。
2. 发现链路入口：Claude Code 的插件发现阶段会先读取 `.claude-plugin/plugin.json`，再扫描组件（`plugins/plugin-dev/skills/plugin-structure/references/component-patterns.md:11-15`，`plugins/plugin-dev/skills/plugin-structure/SKILL.md:343-348`）。
3. 市场分发锚点：marketplace 通过 `source: "./plugins/pr-review-toolkit"` 将该目录纳入可安装插件集合（`.claude-plugin/marketplace.json:117-125`）。

换言之，该文件是“执行逻辑之前的装配层配置”，其有效性决定后续 `commands/` 与 `agents/` 是否能被系统接入。

## 功能点目的

当前 manifest 内容（`plugins/pr-review-toolkit/.claude-plugin/plugin.json:1-9`）：

```json
{
  "name": "pr-review-toolkit",
  "version": "1.0.0",
  "description": "Comprehensive PR review agents specializing in comments, tests, error handling, type design, code quality, and code simplification",
  "author": {
    "name": "Daisy",
    "email": "daisy@anthropic.com"
  }
}
```

各字段目的：

1. `name`
- 插件唯一标识，参与冲突检测与命名空间语义（`plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:15-24`）。
- 必须满足 kebab-case 规则；当前 `pr-review-toolkit` 符合规范（`manifest-reference.md:26-36`）。

2. `version`
- 表达发布版本，遵循语义化版本约定；当前为 `1.0.0`（`plugins/pr-review-toolkit/.claude-plugin/plugin.json:3`，`manifest-reference.md:42-53`）。

3. `description`
- 面向用户/市场展示插件价值主张；当前描述与插件总览文档一致（`plugins/pr-review-toolkit/.claude-plugin/plugin.json:4`，`plugins/README.md:25`）。

4. `author`
- 提供维护归属与联系方式（`plugins/pr-review-toolkit/.claude-plugin/plugin.json:5-8`，`manifest-reference.md:87-114`）。

5. “最小 manifest”策略
- 未配置 `commands/agents/hooks/mcpServers`，说明该插件依赖默认自动发现目录，不在 manifest 中声明自定义路径（`plugins/plugin-dev/skills/plugin-structure/SKILL.md:365-368`，`manifest-reference.md:356-367`）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 关键流程

从插件可见到命令执行的链路：

1. marketplace 注册插件：
- `.claude-plugin/marketplace.json` 中 `pr-review-toolkit` 条目定义 `source`（`.claude-plugin/marketplace.json:117-125`）。

2. 发现阶段读取 manifest：
- 启用插件时，系统读取 `.claude-plugin/plugin.json`（`plugins/plugin-dev/skills/plugin-structure/SKILL.md:343`）。

3. 按默认目录发现组件：
- 扫描 `commands/` 与 `agents/`（`plugins/plugin-dev/skills/plugin-structure/SKILL.md:344-345`；`manifest-reference.md:356-361`）。
- 本插件目录中实际存在 `commands/review-pr.md` 与 6 个 agent 文件（`find plugins/pr-review-toolkit -maxdepth 3 -type f` 结果：9 个文件）。

4. 命令激活与编排：
- 用户调用 `/pr-review-toolkit:review-pr`（`plugins/pr-review-toolkit/commands/review-pr.md:93-95`）。
- 命令工作流先识别变更范围和审查维度，再按条件调度不同 agent，最后聚合结果与行动建议（`review-pr.md:15-88`）。

### 2) 数据结构

A. manifest 数据结构（目标对象）
- 扁平字段：`name`、`version`、`description`
- 嵌套对象：`author.name`、`author.email`

B. 命令协议数据结构（下游）
- `commands/review-pr.md` frontmatter：
  - `description`
  - `argument-hint: "[review-aspects]"`
  - `allowed-tools: ["Bash", "Glob", "Grep", "Read", "Task"]`
  （`plugins/pr-review-toolkit/commands/review-pr.md:1-5`）
- 参数协议：`comments/tests/errors/types/code/simplify/all` + 可选 `parallel`（`review-pr.md:20-29,109-113`）。

C. agent 数据结构（下游）
- 每个 agent markdown 都由 YAML frontmatter + 系统提示正文构成，至少包含 `name/description/model`（如 `code-reviewer.md:1-6`，`comment-analyzer.md:1-6`）。
- 模型配置有 `opus` 与 `inherit` 两类，体现“专职角色+默认继承”的混合策略（`code-reviewer.md:4`，`code-simplifier.md:35`，`comment-analyzer.md:4`，`pr-test-analyzer.md:4`，`silent-failure-hunter.md:4`，`type-design-analyzer.md:4`）。

### 3) 协议与命令

1. 发现协议
- 必须路径：`.claude-plugin/plugin.json`（`manifest-reference.md:7-10`）。
- 加载顺序：默认目录先扫描，再合并 manifest 自定义路径（若存在）；当前插件未覆写路径，因此全部来自默认目录（`manifest-reference.md:352-371`）。

2. 命令编排协议（`review-pr`）
- 审查范围确定：基于 `git diff --name-only` + 可选 `gh pr view`（`review-pr.md:30-33`）。
- 审查维度映射：把变更类型映射到具体 agent（`review-pr.md:37-43`）。
- 执行模式：支持 sequential 与 parallel 两种策略（`review-pr.md:47-55`）。
- 汇总输出协议：固定为 Critical/Important/Suggestions/Strengths + Recommended Action（`review-pr.md:57-88`）。

3. agent 输出协议
- `code-reviewer` 使用 0-100 置信度并只报告 `>=80` 问题（`agents/code-reviewer.md:22-33`）。
- `pr-test-analyzer` 使用 1-10 关键度并按 Critical/Important 分层（`agents/pr-test-analyzer.md:42-57`）。
- `silent-failure-hunter` 定义 CRITICAL/HIGH/MEDIUM 严重性并要求定位、影响、修复建议（`agents/silent-failure-hunter.md:99-109`）。
- `type-design-analyzer` 对 4 个维度做 1-10 评分（`agents/type-design-analyzer.md:24-47,58-69`）。

## 关键代码路径与文件引用

### 目标对象

- `plugins/pr-review-toolkit/.claude-plugin/plugin.json:1-9`

### 调用方（上游）

1. marketplace 路由入口
- `.claude-plugin/marketplace.json:117-125`

2. 插件发现机制与约束
- `plugins/plugin-dev/skills/plugin-structure/references/component-patterns.md:11-17`
- `plugins/plugin-dev/skills/plugin-structure/SKILL.md:343-355`
- `plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:7-10,377-393`

3. 仓库级插件规范文档
- `plugins/README.md:49-60,65-71`

### 被调用方（下游）

1. 命令层
- `plugins/pr-review-toolkit/commands/review-pr.md:1-189`

2. agent 层
- `plugins/pr-review-toolkit/agents/code-reviewer.md:1-47`
- `plugins/pr-review-toolkit/agents/code-simplifier.md:1-83`
- `plugins/pr-review-toolkit/agents/comment-analyzer.md:1-70`
- `plugins/pr-review-toolkit/agents/pr-test-analyzer.md:1-69`
- `plugins/pr-review-toolkit/agents/silent-failure-hunter.md:1-130`
- `plugins/pr-review-toolkit/agents/type-design-analyzer.md:1-110`

### 配置、测试、脚本、文档上下文

1. 配置
- 插件 manifest：`plugins/pr-review-toolkit/.claude-plugin/plugin.json`
- 市场聚合配置：`.claude-plugin/marketplace.json:117-125`
- 命令执行配置（frontmatter + allowed-tools）：`plugins/pr-review-toolkit/commands/review-pr.md:1-5`

2. 测试
- 插件目录未发现测试文件模式（`.test/.spec/__tests__/tests`），当前更偏“提示词工作流”而非可执行测试套件（`find + rg` 检索结果为空）。

3. 脚本
- 插件目录未发现 `*.sh/*.py/*.js/*.ts` 运行脚本（`find + rg` 检索结果为空）；流程逻辑由命令/agent 文本驱动。

4. 文档
- 插件内主文档：`plugins/pr-review-toolkit/README.md:1-313`
- 仓库插件索引：`plugins/README.md:25`
- 仓库根入口说明：`README.md:48-50`

## 依赖与外部交互

### 仓库内依赖

1. 对 marketplace `source` 路径正确性的依赖
- 若 `source` 偏移，manifest 不会被正确发现（`.claude-plugin/marketplace.json:124`）。

2. 对默认自动发现机制的依赖
- 当前 manifest 未声明自定义路径，强依赖默认目录扫描（`SKILL.md:343-348`，`manifest-reference.md:356-361`）。

3. 对命令与 agent 文本质量的依赖
- 该插件核心行为来自 markdown 协议（workflow 指令、评分标准、输出格式），而非脚本代码。

### 外部交互（间接）

`plugin.json` 本身不访问外部系统，但其下游命令明确依赖：

1. Git/GitHub CLI 语义
- `git diff --name-only`、`gh pr view` 被用于识别 PR 语境（`review-pr.md:31-33`）。

2. Claude Task 调度
- 通过 `Task` 工具调起不同 agent（`review-pr.md:4` + 各 agent frontmatter）。

3. 运行时仓库上下文
- 多个 agent 默认围绕“近期改动 / git diff / PR 变更”审查（如 `code-reviewer.md:12`，`pr-test-analyzer.md:35-40`）。

## 风险、边界与改进建议

### 风险

1. 元数据双源漂移
- `plugin.json` 与 `.claude-plugin/marketplace.json` 同时维护 `name/description/version/author`；当前 author 已不一致：
  - manifest：`Daisy`（`plugins/pr-review-toolkit/.claude-plugin/plugin.json:6-7`）
  - marketplace：`Anthropic`（`.claude-plugin/marketplace.json:120-123`）
- 这会导致展示归属、维护责任、后续升级信息的歧义。

2. 文档承诺与执行边界存在“软约束”
- `review-pr.md` 描述了完整流程，但本质为自然语言执行协议，不是强制脚本；实际质量依赖执行模型是否严格遵循步骤。

3. 缺乏自动化回归保护
- 插件无测试与脚本校验，命令/agent 的 frontmatter 漂移或提示词退化难以及时发现。

4. 外部命令环境依赖
- 流程假设 `gh` 可用且认证正确（`review-pr.md:32`），否则 PR 相关能力会降级。

### 边界

1. `plugin.json` 只负责发现与元数据，不负责业务审查正确性。
2. 该插件不直接定义 hook/MCP；与外部系统交互主要发生在命令执行阶段。
3. 审查质量上限由 agent 提示词设计与调用时上下文质量共同决定。

### 改进建议

1. 增加 manifest 与 marketplace 一致性检查
- 在 CI 增加 `name/version/description/author` 对齐校验，避免双源漂移。

2. 增加插件静态 lint/smoke
- 校验项建议：manifest JSON、命令 frontmatter 字段、agent frontmatter 完整性、关键 workflow 段落存在性。

3. 为 `review-pr` 补充失败退化策略
- 在命令文档中明确 `gh pr view` 不可用时的 fallback（例如仅基于本地 diff）。

4. 统一作者信息来源
- 明确以 manifest 还是 marketplace 为 canonical source，并在发布流程中自动回填另一处。
