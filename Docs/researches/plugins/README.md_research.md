# FILE `plugins/README.md` 研究文档

## 场景与职责

`plugins/README.md` 是仓库内“插件子生态入口文档”，核心职责不是执行逻辑，而是把插件能力地图、安装路径和结构约定交付给读者与维护者。

1. 目录级导航职责
- 通过插件总表把 13 个插件的能力暴露为“名称 + 描述 + 组件摘要”（`plugins/README.md:11-27`）。
- 作为根 README 的下游入口被显式链接（`README.md:48-50`）。

2. 插件模型说明职责
- 解释 Claude Code 插件的能力面：commands、agents、hooks、skills、MCP（`plugins/README.md:5-9`）。

3. 结构规范职责
- 给出标准目录骨架（`.claude-plugin/plugin.json`、`commands/`、`agents/`、`skills/`、`hooks/`、`.mcp.json`、`README.md`）（`plugins/README.md:47-61`）。

4. 贡献约束职责
- 对新增插件提出“有 README、有 manifest、有命令/agent 文档”的维护规范（`plugins/README.md:63-71`）。

5. 在插件体系中的位置（调用方/被调用方）
- 调用方（上游）：根 README 的 Plugins 段（`README.md:48-50`）。
- 被调用方（下游）：各插件目录链接（`plugins/README.md:15-27`）与外部官方文档链接（`plugins/README.md:9,45,75-77`）。

## 功能点目的

1. 插件能力总览
- 目的：让用户在一个页面快速判断“哪个插件解决什么问题”。
- 实现：Markdown 表格按插件列出命令/agent/skill/hook（`plugins/README.md:13-27`）。

2. 安装与启用指引
- 目的：给首次使用者最短操作路径。
- 实现：`npm install -g`、`claude`、`/plugin` 或 `.claude/settings.json` 配置（`plugins/README.md:29-45`）。

3. 统一结构约定
- 目的：降低新增插件的结构分歧与加载不确定性。
- 实现：以目录树形式定义“标准插件结构”（`plugins/README.md:49-61`）。

4. 贡献检查点
- 目的：使插件文档与元数据可维护、可发现。
- 实现：5 条贡献要求（`plugins/README.md:65-71`）。

5. 文档分层目的
- 顶层目录文档负责“索引与约定”（本文件），每个插件 `README.md` 负责“插件内部细节”，`plugin.json` 负责“机器可读元数据”（如 `plugins/code-review/.claude-plugin/plugin.json:1-9`、`plugins/feature-dev/.claude-plugin/plugin.json:1-9`）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 目录导航流程（文档流）
1. 用户从仓库根文档进入插件入口（`README.md:48-50`）。
2. 在 `plugins/README.md` 中按表格挑选插件（`plugins/README.md:13-27`）。
3. 跳转到目标插件目录阅读具体 README/commands/agents/skills/hooks。
4. 如需安装，继续按 `/plugin` 或 settings 配置（`plugins/README.md:43-45`）。

该流程是“文档驱动发现”，不是脚本驱动发现。

### 2) 数据结构与接口约定
1. 索引数据结构：Markdown 表格
- 行粒度：单插件。
- 列粒度：`Name | Description | Contents`（`plugins/README.md:13-14`）。
- `Contents` 以“组件类型 + 标识符”形式表达（命令名、agent 名、skill 名、hook 事件）。

2. 插件元数据结构：`plugin.json`
- 仓库中的插件 manifest 以 `name/version/description/author` 为主（示例：`plugins/commit-commands/.claude-plugin/plugin.json:1-9`）。

3. 市场注册结构：`.claude-plugin/marketplace.json`
- 使用 `plugins[].source` 指向 `./plugins/<name>`（`.claude-plugin/marketplace.json:10-149`）。
- 该文件是目录级插件分发配置，与本 README 形成“机器注册 + 人类索引”双轨。

### 3) 协议层实现（与本 README 的下游契约）
本 README 在 `Plugin Structure` 小节声明的目录约定，与各组件协议对应：

1. Command 协议
- Markdown + YAML frontmatter（`description`、`argument-hint`、`allowed-tools`），如 `plugins/commit-commands/commands/commit.md:1-4`。

2. Hook 协议
- `hooks/hooks.json` 按事件注册 command hook，如 `PreToolUse`/`Stop`/`SessionStart`（`plugins/hookify/hooks/hooks.json:1-49`、`plugins/ralph-wiggum/hooks/hooks.json:1-15`）。

3. Skill 协议
- `skills/<skill>/SKILL.md`，frontmatter 至少含 `name/description`，如 `plugins/frontend-design/skills/frontend-design/SKILL.md:1-4`。

4. Agent 协议
- `agents/*.md` 使用 frontmatter + system prompt（规范详见 `plugins/plugin-dev/skills/agent-development/SKILL.md:20-45`）。

5. Hook 脚本执行协议（代表实现）
- `hookify` 通过 Python 读取 stdin JSON、加载规则、输出 JSON 决策（`plugins/hookify/hooks/pretooluse.py:35-70`）。
- 规则数据结构由 `Rule/Condition` dataclass 承载（`plugins/hookify/core/config_loader.py:15-43`），匹配引擎在 `RuleEngine` 中统一求值（`plugins/hookify/core/rule_engine.py:35-94`）。

### 4) 关键命令与流程实例（证明“Contents”不是静态文案）
1. `code-review`：命令实际编排并行评审与验证流程（`plugins/code-review/commands/code-review.md:30-57`）。
2. `feature-dev`：7 阶段流程编排（`plugins/feature-dev/commands/feature-dev.md:20-80`）。
3. `plugin-dev`：8 阶段插件开发流程（`plugins/plugin-dev/commands/create-plugin.md:233-341`）。
4. `ralph-wiggum`：`/ralph-loop` 调起 `setup-ralph-loop.sh`，Stop Hook 读取 transcript 并阻断退出（`plugins/ralph-wiggum/commands/ralph-loop.md:1-17`、`plugins/ralph-wiggum/scripts/setup-ralph-loop.sh:130-150`、`plugins/ralph-wiggum/hooks/stop-hook.sh:57-175`）。

## 关键代码路径与文件引用

1. 目标文件与直接上游
- `plugins/README.md:1-77`
- `README.md:48-50`

2. 机器注册与目录源
- `.claude-plugin/marketplace.json:10-149`

3. 结构规范的“细则来源”
- `plugins/README.md:47-61`（总纲）
- `plugins/plugin-dev/skills/plugin-structure/SKILL.md:22-45`（manifest 与目录细则）

4. 与目录表项对应的代表性实现路径
- 命令：`plugins/code-review/commands/code-review.md:1-109`
- Hook 配置：`plugins/hookify/hooks/hooks.json:1-49`
- Hook 实现：`plugins/hookify/hooks/pretooluse.py:35-70`
- 规则结构：`plugins/hookify/core/config_loader.py:15-84`
- 规则引擎：`plugins/hookify/core/rule_engine.py:35-94`
- Stop Hook 循环：`plugins/ralph-wiggum/hooks/stop-hook.sh:165-174`
- 安全拦截：`plugins/security-guidance/hooks/security_reminder_hook.py:31-126`

5. 配置/测试/脚本上下文
- 插件开发验证脚本入口（用于约束 README 所声明结构）：
  - `plugins/plugin-dev/skills/agent-development/scripts/validate-agent.sh`
  - `plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh`
  - `plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh`
  - `plugins/plugin-dev/skills/plugin-settings/scripts/validate-settings.sh`

## 依赖与外部交互

1. 外部文档依赖
- `docs.claude.com` 插件文档与 Agent SDK 文档（`plugins/README.md:9,45,75-77`）。

2. CLI/生态依赖（由下游插件实现承担）
- `git/gh`：如 `code-review`、`commit-commands`（`plugins/code-review/commands/code-review.md:2`，`plugins/commit-commands/commands/commit-push-pr.md:2`）。
- `python3`：hookify/security-guidance hook（`plugins/hookify/hooks/hooks.json:9-43`、`plugins/security-guidance/hooks/hooks.json:9-13`）。
- `jq/awk/sed/perl`：Ralph Stop Hook 与脚本链路（`plugins/ralph-wiggum/hooks/stop-hook.sh:57-170`）。

3. 与配置系统的交互
- marketplace 注册：`.claude-plugin/marketplace.json:10-149`。
- 每插件元数据：`plugins/*/.claude-plugin/plugin.json`。
- Hook 事件配置：`plugins/*/hooks/hooks.json`。

4. 与研究流程脚本交互（本次任务上下文）
- checklist/todo 生成脚本通过 `Docs/researches/blueprint_checklist.md` 统计研究进度（`.ops/generate_daily_research_todo.sh:5-39`）。

5. 测试边界
- `plugins/README.md` 本身无自动化单测；当前校验主要依赖人工审阅与 plugin-dev 辅助脚本，且脚本聚焦组件格式而非目录索引文案一致性。

## 风险、边界与改进建议

1. 风险：安装指引与根 README 冲突
- 根 README 标注 npm 安装“已弃用”（`README.md:15,41-44`），但 `plugins/README.md` 仍把 `npm install -g` 作为安装步骤（`plugins/README.md:33-36`）。
- 建议：改为与根 README 一致的推荐安装方式，并把 npm 迁到“兼容/历史方式”。

2. 风险：目录结构声明与现状存在偏差
- 本文件声明“Each plugin follows standard structure”（`plugins/README.md:49`），且贡献要求要求 `plugin.json` 与 `README.md`（`plugins/README.md:67-70`）。
- 但当前 `plugins/plugin-dev` 缺 `.claude-plugin/plugin.json`；`plugins/security-guidance` 缺 `README.md`（目录扫描结果）。
- 建议：
  - 方案 A：补齐缺失文件。
  - 方案 B：在本文件明确“示例插件允许的例外”并列出例外名单。

3. 风险：能力描述与真实实现漂移
- `plugins/README.md` 将 `code-review` 描述为“5 parallel Sonnet agents”（`plugins/README.md:17`），而命令定义实际是“4 个并行 agent”，并含 Opus 角色（`plugins/code-review/commands/code-review.md:30-39`）。
- 建议：把目录表“Contents”改成可由命令文件头部元数据自动生成，减少手工漂移。

4. 风险：人工维护成本高，缺少一致性检查
- 目前未见脚本校验“README 索引表 ↔ 插件目录实际组件数量/类型”一致性。
- 建议新增 lint：
  - 校验每一行插件是否存在。
  - 校验 `Contents` 中声明的命令/agent/skill/hook 与实际文件匹配。
  - 校验结构段落要求与目录实际（manifest/readme）一致。

5. 边界说明
- 本文件是“文档编排层”，不会直接被运行时执行；真实行为由下游命令/agent/hook/skill 文件决定。
- 因此该文件的主要质量指标不是“可执行正确性”，而是“索引准确性、约束一致性、跳转有效性”。

6. 可执行改进路线（建议优先级）
1. 先修安装指引冲突（低成本高收益）。
2. 再修 `code-review` 描述偏差与结构例外（避免误导）。
3. 最后补一条 README 索引一致性校验脚本接入 CI（长期防漂移）。
