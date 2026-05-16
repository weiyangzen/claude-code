# plugins/code-review/.claude-plugin/plugin.json 研究

## 场景与职责

`plugins/code-review/.claude-plugin/plugin.json` 是 `code-review` 插件的 manifest（`plugins/code-review/.claude-plugin/plugin.json:1-9`）。
它本身不执行 PR 审查逻辑，而是插件可发现、可加载、可展示的入口配置。

在仓库中的职责定位：
1. 声明插件身份（`name/version/description/author`），用于插件识别与归属。
2. 作为插件发现链路的起点文件，支撑后续 `commands/` 组件自动加载。
3. 与 marketplace 条目一起构成安装与展示元数据闭环。

上下文依据：
- 插件标准结构要求 `.claude-plugin/plugin.json`：`plugins/README.md:49-60`
- manifest 必须位于该路径，否则无法识别：`plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:7-10`
- 发现阶段先读取各插件 manifest，再注册组件：`plugins/plugin-dev/skills/plugin-structure/references/component-patterns.md:11-15`

## 功能点目的

1. 唯一标识与冲突规避
- `name: "code-review"` 是插件唯一标识关键字段（`plugins/code-review/.claude-plugin/plugin.json:2`）。
- `name` 的语义是识别/冲突检测/可选命名空间（`plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:15-24`）。

2. 发布版本与归属
- `version: "1.0.0"`、`author.name/email` 提供发布与责任归属信息（`plugins/code-review/.claude-plugin/plugin.json:4-8`）。
- `version` 推荐语义化版本（`plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:42-63`）。

3. 能力描述（供市场与用户理解）
- `description` 用一句话定义该插件价值（`plugins/code-review/.claude-plugin/plugin.json:3`）。
- marketplace 同步展示该描述与作者信息（`.claude-plugin/marketplace.json:29-37`）。

4. 保持 manifest 精简，依赖默认自动发现
- 当前 manifest 未声明 `commands/agents/hooks/mcpServers` 自定义路径（`plugins/code-review/.claude-plugin/plugin.json:1-9`）。
- 因此走默认发现：扫描 `commands/`、`agents/`、`skills/` 等（`plugins/plugin-dev/skills/plugin-structure/SKILL.md:343-348`；`plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:356-366`）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 关键流程（调用链）

1. marketplace 注册 `code-review`，`source` 指向插件目录：`.claude-plugin/marketplace.json:29-37`
2. Claude Code 在发现阶段读取该目录下 `.claude-plugin/plugin.json`：`plugins/plugin-dev/skills/plugin-structure/references/component-patterns.md:11-15`
3. 组件自动发现 `commands/code-review.md`，注册 `/code-review`：`plugins/plugin-dev/skills/plugin-structure/SKILL.md:343-345`
4. 用户触发 `/code-review` 后进入命令协议，执行 PR 审查与评论：`plugins/code-review/commands/code-review.md:1-109`

### 2) 数据结构（manifest）

当前对象结构：
- `name`：`code-review`（`plugins/code-review/.claude-plugin/plugin.json:2`）
- `description`：功能描述（`plugins/code-review/.claude-plugin/plugin.json:3`）
- `version`：`1.0.0`（`plugins/code-review/.claude-plugin/plugin.json:4`）
- `author`：`name/email`（`plugins/code-review/.claude-plugin/plugin.json:5-8`）

实现要点：
- 这是“最小可用 manifest + 完整元数据”形态。
- 未定义任何路径扩展字段，符合“lean manifest”实践（`plugins/plugin-dev/skills/plugin-structure/SKILL.md:365-368`）。

### 3) 协议与命令耦合点

虽然 `plugin.json` 不写执行步骤，但它决定命令文件是否可加载，间接决定以下协议能否生效：
- 命令工具白名单：`allowed-tools` 限定 `gh` 子命令和 `mcp__github_inline_comment__create_inline_comment`（`plugins/code-review/commands/code-review.md:1-3`）。
- 审查流程协议：跳过判断 -> 收集 CLAUDE.md -> 并行审查 -> 问题验证 -> 终端/评论输出（`plugins/code-review/commands/code-review.md:14-77`）。
- 评论链接协议：要求 full SHA + `#Lx-Ly` 行区间（`plugins/code-review/commands/code-review.md:103-109`）。

### 4) 被调用方（运行时）

`plugin.json` 载入后激活的命令会调用：
- GitHub CLI：`gh pr view/diff/list/comment` 等（`plugins/code-review/commands/code-review.md:2,18,65,90`）
- MCP 工具：`mcp__github_inline_comment__create_inline_comment`（`plugins/code-review/commands/code-review.md:2,71`）

## 关键代码路径与文件引用

核心对象：
- `plugins/code-review/.claude-plugin/plugin.json:1-9`

上游调用方与注册路径：
- `.claude-plugin/marketplace.json:29-37`（marketplace 注册与 source 路由）
- `plugins/plugin-dev/skills/plugin-structure/references/component-patterns.md:11-15`（发现顺序）
- `plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:7-10`（manifest 必需位置）
- `plugins/README.md:49-60`（标准目录结构）

下游被调用方与实现体：
- `plugins/code-review/commands/code-review.md:1-109`（执行协议）
- `plugins/code-review/README.md:11-258`（用户面文档）

配置、测试、脚本、文档：
- 配置：`plugin.json` 元数据 + marketplace 元数据 + command frontmatter 工具白名单。
- 测试：`plugins/code-review` 目录未包含测试文件（仅 manifest、README、command 三个文件）。
- 脚本：`plugins/code-review` 目录未包含脚本文件。
- 文档：插件内 `README.md` + 仓库级 `plugins/README.md` + 根 `README.md` 插件入口说明（`README.md:48-50`）。

## 依赖与外部交互

仓库内依赖：
1. 依赖 `.claude-plugin/marketplace.json` 的 `source` 正确指向插件根（`.claude-plugin/marketplace.json:36`）。
2. 依赖 Claude Code 插件发现机制先读 manifest，再扫描组件（`plugins/plugin-dev/skills/plugin-structure/references/component-patterns.md:11-15`）。
3. 依赖默认目录自动发现规则（`plugins/plugin-dev/skills/plugin-structure/SKILL.md:343-348`）。

外部交互（经下游命令发生）：
1. 与 GitHub 交互：读取 PR、读取 diff、写评论（`plugins/code-review/commands/code-review.md:2,65,90`）。
2. 与 MCP 服务交互：逐条 inline comment（`plugins/code-review/commands/code-review.md:71`）。
3. 与仓库内容交互：读取相关 `CLAUDE.md` 路径作为审查规则输入（`plugins/code-review/commands/code-review.md:24-27,33`）。

## 风险、边界与改进建议

风险：
1. 元数据双源漂移风险
- `plugin.json` 与 `marketplace.json` 重复维护 `name/description/version/author`，长期存在不一致可能（`plugins/code-review/.claude-plugin/plugin.json:2-8`；`.claude-plugin/marketplace.json:29-35`）。

2. 文档与执行协议漂移风险
- `README` 描述“0-100 评分 + 80 阈值过滤”（`plugins/code-review/README.md:23-25,78-83,216-221,239-243`）。
- 命令正文强调“问题验证后过滤”，未显式定义分数计算步骤（`plugins/code-review/commands/code-review.md:55-57`）。

3. 能力描述漂移风险
- `plugins/README.md` 将该插件描述为“5 parallel Sonnet agents”（`plugins/README.md:17`）。
- 命令文件实际是“4 agents in parallel”，且包含 Opus 角色（`plugins/code-review/commands/code-review.md:30,35,38`）。

4. 外部依赖链路脆弱性
- `gh` 权限/登录态或 MCP 服务异常会影响 `--comment` 闭环。

边界：
1. `plugin.json` 只负责声明，不负责审查质量与结果正确性。
2. `plugin.json` 不直接定义工具调用细节，细节在 `commands/code-review.md`。
3. 该插件目录无自带测试/脚本，行为一致性主要依赖文档与运行时实践。

改进建议：
1. 加入 manifest 与 marketplace 一致性校验脚本（CI 对齐关键字段）。
2. 在 `commands/code-review.md` 中显式补充分数模型，或同步收敛 README 叙述，避免认知偏差。
3. 修正 `plugins/README.md` 中 `code-review` 的 agent 数量/模型描述，确保目录索引与实现一致。
4. 为 `plugins/code-review` 增加最小回归检查（例如 frontmatter 合法性、关键步骤存在性、评论链接格式规则存在性）。
