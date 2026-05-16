# plugins/code-review/.claude-plugin 目录研究（DIR）

## 场景与职责

`plugins/code-review/.claude-plugin` 是 `code-review` 插件的 manifest 目录，当前仅包含 `plugin.json`（`plugins/code-review/.claude-plugin/plugin.json:1-9`）。

该目录不承载审查算法或命令执行逻辑，而是 Claude Code 插件生命周期中的“发现与身份入口”：
- 插件结构规范要求 manifest 必须位于 `.claude-plugin/plugin.json`，否则插件不会被识别（`plugins/README.md:49-55`，`plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:7-10`）。
- 组件生命周期中，启动阶段会先扫描已启用插件并读取该 manifest，再进行组件发现与注册（`plugins/plugin-dev/skills/plugin-structure/references/component-patterns.md:9-16`）。
- 仓库 marketplace 通过 `source: "./plugins/code-review"` 将插件根目录接入发现链路（`.claude-plugin/marketplace.json:29-37`）。

因此，本目录职责可归纳为：
1. 提供插件唯一标识与元信息。
2. 作为命令组件（`commands/code-review.md`）可被自动发现的前置条件。
3. 作为 marketplace 安装/展示元数据链路中的一致性锚点。

## 功能点目的

### 1) 插件身份声明与唯一性
`plugin.json` 声明：
- `name: "code-review"`
- `description`
- `version: "1.0.0"`
- `author.name/email`
见 `plugins/code-review/.claude-plugin/plugin.json:2-8`。

其中 `name` 是核心字段，用于插件识别、冲突检测与命名语义（`plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:15-31`）。

### 2) marketplace 到插件实体的路由闭环
- marketplace 条目声明插件来源路径（`.claude-plugin/marketplace.json:36`）。
- 运行时在该来源路径内读取 `.claude-plugin/plugin.json` 完成实例化。

这意味着 marketplace 的“source 路径正确性”和本目录 manifest 的“结构正确性”共同决定插件是否可用。

### 3) 默认组件发现策略的隐式确认
本 manifest 未声明 `commands/agents/hooks/mcpServers` 自定义路径字段（`plugins/code-review/.claude-plugin/plugin.json:1-9`），因此依赖默认目录发现机制（`plugins/plugin-dev/skills/plugin-structure/SKILL.md:86-115`）：
- `commands/` 下 markdown 自动作为 slash command 加载。
- 对本插件而言，实际能力入口落在 `plugins/code-review/commands/code-review.md`。

### 4) 发布与归属信息承载
manifest 与 marketplace 同时保留作者和版本信息（`plugins/code-review/.claude-plugin/plugin.json:4-8`；`.claude-plugin/marketplace.json:31-35`），用于展示、归因和版本管理。

## 具体技术实现（关键流程/数据结构/协议/命令）

### A. 关键流程（调用链）

1. 上游调用方：插件市场与发现器
- `.claude-plugin/marketplace.json` 注册 `code-review` 并指向 `./plugins/code-review`（`.claude-plugin/marketplace.json:29-37`）。
- Claude Code 初始化阶段扫描插件并读取 `.claude-plugin/plugin.json`（`plugins/plugin-dev/skills/plugin-structure/references/component-patterns.md:9-16`）。

2. 目标对象：本目录 manifest
- `plugins/code-review/.claude-plugin/plugin.json` 提供插件元数据（`plugins/code-review/.claude-plugin/plugin.json:1-9`）。

3. 下游被调用方：命令组件与外部工具
- 组件发现后，`/code-review` 对应 `plugins/code-review/commands/code-review.md`（`plugins/code-review/commands/code-review.md:1-4`）。
- 命令执行时调用 `gh` 子命令与 `mcp__github_inline_comment__create_inline_comment`（`plugins/code-review/commands/code-review.md:2,71,90`）。

简化链路：
`marketplace(source)` -> `plugin root` -> `.claude-plugin/plugin.json` -> `commands/` 自动发现 -> `/code-review` 执行审查与评论流程。

### B. 数据结构

本目录核心数据结构是最小 manifest JSON：

```json
{
  "name": "code-review",
  "description": "Automated code review for pull requests using multiple specialized agents with confidence-based scoring",
  "version": "1.0.0",
  "author": {
    "name": "Boris Cherny",
    "email": "boris@anthropic.com"
  }
}
```

字段含义：
- `name`：唯一标识（必需，kebab-case，`manifest-reference.md:15-40`）。
- `description`：用途摘要（`manifest-reference.md:65-83`）。
- `version`：语义化版本（`manifest-reference.md:42-63`）。
- `author`：作者归属与联系方式（`manifest-reference.md:87-114`）。

实现特征：
- 该 manifest 仅含元数据，不含执行配置（如 `mcpServers`、自定义 commands path）。
- 这使目录保持“声明性最小化”，但运行行为完全依赖下游 markdown 指令。

### C. 协议与命令约束（与本目录强相关）

虽然 `.claude-plugin` 不直接执行命令，但它决定命令能否被加载，进而决定以下协议是否生效：
- 命令 frontmatter 的工具白名单（`allowed-tools`）控制 `gh` 与 MCP 使用面（`plugins/code-review/commands/code-review.md:1-4`）。
- 审查流程协议：跳过条件、并行审查、问题验证、`--comment` 分支（`plugins/code-review/commands/code-review.md:14-77`）。
- 评论协议：inline comment 要求 `confirmed: true`，且链接格式必须 full SHA + `#Lx-Ly`（`plugins/code-review/commands/code-review.md:71,103-109`）。

也就是说，本目录是“协议入口开关”，命令文件是“协议执行体”。

## 关键代码路径与文件引用

### 目标目录直接对象
- `plugins/code-review/.claude-plugin/plugin.json:1-9`

### 调用方（上游）
- `.claude-plugin/marketplace.json:29-37`（插件注册与 source 路径）
- `plugins/plugin-dev/skills/plugin-structure/references/component-patterns.md:9-16`（启动发现顺序）
- `plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:7-10`（manifest 必须路径）
- `plugins/README.md:49-61`（标准插件结构）

### 被调用方（下游）
- `plugins/code-review/commands/code-review.md:1-109`（实际运行协议与外部调用）
- `plugins/code-review/README.md:11-258`（功能说明、操作、排障、配置说明）

### 配置、测试、脚本、文档上下文
- 配置：manifest 元数据 + marketplace 注册项 + command frontmatter（`plugins/code-review/.claude-plugin/plugin.json:1-9`，`.claude-plugin/marketplace.json:29-37`，`plugins/code-review/commands/code-review.md:1-4`）。
- 测试：`plugins/code-review` 下未发现 `test/spec` 文件（目录内仅 3 个文件：manifest、README、command）。
- 脚本：目标目录与插件目录均无 `scripts/` 执行脚本。
- 文档：`plugins/code-review/README.md` 提供完整用户文档；`plugins/README.md` 提供插件索引。

## 依赖与外部交互

### 1) 仓库内依赖
- 依赖 marketplace 将插件目录纳入候选（`.claude-plugin/marketplace.json:29-37`）。
- 依赖插件发现机制正确读取 manifest（`component-patterns.md:11-14`）。
- 依赖默认 `commands/` 自动发现规则加载 `/code-review`（`plugin-structure/SKILL.md:112-115`）。

### 2) 外部交互（间接，由下游命令触发）
本目录本身不直接联网或执行工具；外部交互来自被其“放行加载”的 `/code-review`：
- GitHub CLI：读取 PR、评论、diff、发摘要评论（`plugins/code-review/commands/code-review.md:2,18,65,90`）。
- MCP：逐条发送 inline comment（`plugins/code-review/commands/code-review.md:2,71`）。
- 审查上下文读取本仓库 `CLAUDE.md`（若存在）和 PR 改动路径（`plugins/code-review/commands/code-review.md:24-27,33`）。

### 3) 一致性关系
`plugin.json` 与 `marketplace.json` 在 `name/description/version/author` 上存在双源信息（`plugins/code-review/.claude-plugin/plugin.json:2-8`；`.claude-plugin/marketplace.json:29-35`），需保持同步以避免展示或追责信息漂移。

## 风险、边界与改进建议

### 风险
1. 单点失效风险
- 目录中只有 `plugin.json`；一旦 JSON 语法损坏或路径错误，插件会在发现阶段失效。

2. 元数据双源漂移
- `plugin.json` 与 `marketplace.json` 重复维护元信息，长期存在不一致风险。

3. 自动化校验不足
- 目前未见针对 `plugins/*/.claude-plugin/plugin.json` 的仓库级强制 schema 校验；更多依赖人工或运行时暴露问题。

4. 文档承诺与执行协议漂移会被 manifest 放大
- 本目录本身不执行业务逻辑，但其下游 `README` 与命令指令已存在语义差异（如 README 强调“0-100 评分 + 80 阈值”，而命令文本主要是“验证通过后保留”流程，`plugins/code-review/README.md:23-25,216-219` vs `plugins/code-review/commands/code-review.md:55-57`）。manifest 只要生效，就会把这类差异直接带入运行时行为预期。

### 边界
1. 本目录不定义 `allowed-tools`、模型、审查步骤，也不直接调用外部系统。
2. 本目录只负责元数据声明与可发现性，不负责审查结果质量。
3. 目录内无测试与脚本，不能独立完成行为回归验证。

### 改进建议
1. 增加 manifest CI 校验
- 对 `plugins/*/.claude-plugin/plugin.json` 执行 JSON schema + `name/version/author` lint（可参考 `plugin-validator` 的校验项，`plugins/plugin-dev/agents/plugin-validator.md:56-66`）。

2. 增加 marketplace 与 manifest 一致性检查
- 在 CI 中自动比对 `source` 所指插件 manifest 的关键字段，避免双源漂移。

3. 为 code-review 插件补充“文档-命令一致性检查”
- 至少对阈值策略、历史分析职责、输出行为做文本规则校验，减少用户认知偏差。

4. 适度补充 manifest 元数据
- 可考虑加入 `homepage/repository/license/keywords`，提升市场可发现性和维护可追踪性（字段定义见 `manifest-reference.md:115-207`）。
