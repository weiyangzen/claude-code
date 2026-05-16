# plugins/plugin-dev/commands 目录研究（DIR）

## 场景与职责

`plugins/plugin-dev/commands` 是 `plugin-dev` 插件的“可执行工作流入口层”。当前目录仅包含 1 个命令文件：

- `plugins/plugin-dev/commands/create-plugin.md`

该文件定义 `/plugin-dev:create-plugin`，把“插件从需求到发布”的全过程编排为多阶段流程，由 Claude 在命令触发后按提示执行，而不是传统脚本直接执行（`plugins/plugin-dev/commands/create-plugin.md:1-5,7-10`）。

在仓库级上下文中，它承担以下职责：

- 作为 `plugin-dev` 的核心能力入口（`plugins/plugin-dev/README.md:19-33`）。
- 在插件总览中代表 `plugin-dev` 暴露的唯一 command 能力（`plugins/README.md:24`）。
- 与 marketplace 元数据中的 `plugin-dev` 插件源路径配合，被安装后以 namespaced 命令形式提供（`/plugin-dev:create-plugin`，`plugins/plugin-dev/README.md:21,45`；`.claude-plugin/marketplace.json:106-114`）。

## 功能点目的

### 1) 提供结构化的插件创建向导

该命令把复杂的插件开发活动拆为 8 个阶段：Discovery、Component Planning、Detailed Design、Structure Creation、Component Implementation、Validation、Testing、Documentation（`plugins/plugin-dev/commands/create-plugin.md:24-341`，`plugins/plugin-dev/README.md:25-33`）。

目的：

- 强制先澄清需求与边界，避免“直接开写”导致返工（`plugins/plugin-dev/commands/create-plugin.md:13,83,94-99`）。
- 把技能加载、代理调用、验证步骤前置为流程要求，而不是“可选建议”（`plugins/plugin-dev/commands/create-plugin.md:48,157-164,238-260,371-375`）。

### 2) 统一组件化插件开发方法

命令在 Phase 2/5 明确引导用户决定是否需要 skills/commands/agents/hooks/MCP/settings，并逐类实现（`plugins/plugin-dev/commands/create-plugin.md:52-58,167-225`）。

目的：

- 让插件组件选择显式化，减少遗漏关键组件（如 hooks、settings）。
- 复用 `plugin-dev` 内置技能体系（7 个 skills）和 3 个 agents，形成“规划 -> 实现 -> 复核”的闭环（`plugins/plugin-dev/README.md:7-15,35-41`）。

### 3) 质量关口与交付关口内建

Phase 6/7/8 分别覆盖校验、实测、文档收尾（`plugins/plugin-dev/commands/create-plugin.md:233-341`）。

目的：

- 在用户真正发布前做结构化质量检查，而不是仅凭主观判断。
- 把测试方式（`cc --plugin-dir`、`/help`、`/mcp` 等）作为标准交付路径的一部分（`plugins/plugin-dev/commands/create-plugin.md:278-299`）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### A. 命令协议与执行模型

`create-plugin.md` 使用 slash command 的 Markdown + YAML frontmatter 协议：

- `description`：命令说明（help 展示语义）
- `argument-hint`：参数提示
- `allowed-tools`：显式放行工具集合

定义见 `plugins/plugin-dev/commands/create-plugin.md:1-5`；字段语义见 `plugins/plugin-dev/skills/command-development/references/frontmatter-reference.md:24-67,60-66,196-203`。

该命令将 `$ARGUMENTS` 作为初始需求输入（`plugins/plugin-dev/commands/create-plugin.md:20`），符合命令参数替换约定（`plugins/plugin-dev/skills/command-development/SKILL.md:195-221`）。

### B. 关键流程（调用方 -> 本目录 -> 被调用方）

1. 用户调用 `/plugin-dev:create-plugin [optional description]`（`plugins/plugin-dev/README.md:43-50`）。
2. Claude 读取本命令内容，按阶段执行并使用 `TodoWrite` 持续记录（`plugins/plugin-dev/commands/create-plugin.md:18,349`）。
3. 在阶段节点调用内置工具：
- `AskUserQuestion`：收集需求和决策确认（`plugins/plugin-dev/commands/create-plugin.md:33-38,94,267,300,363-369`）。
- `Skill`：加载 `plugin-structure`、`command-development`、`agent-development` 等技能（`plugins/plugin-dev/commands/create-plugin.md:48,157-164,373-375`）。
- `Task`：调用 `agent-creator`、`plugin-validator`、`skill-reviewer`（`plugins/plugin-dev/commands/create-plugin.md:15,180,194-200,238-251,351`）。
- `Bash/Write/Read/Grep/Glob`：创建目录、写 manifest、检查文件与执行脚本（`plugins/plugin-dev/commands/create-plugin.md:125-147,133-144,199,258-259`）。

### C. 目录内唯一命令的“编排 DSL”特征

虽然是 Markdown 文件，但其本质是“Claude 任务编排协议”：

- 具备阶段化状态机：每阶段有 Goal/Actions/Output。
- 具备人工确认闸门：多个关键决策点要求等待用户确认（`plugins/plugin-dev/commands/create-plugin.md:363-369`）。
- 具备组件能力映射：每类组件对应必须加载的 skill（`plugins/plugin-dev/commands/create-plugin.md:157-164`）。

### D. 关联脚本与验证命令（命令内引用）

本目录不直接存放 shell 脚本，但 `create-plugin.md` 明确把以下脚本纳入流程：

- `validate-agent.sh`（agent 文件结构校验，`plugins/plugin-dev/commands/create-plugin.md:199,255`；脚本定义 `plugins/plugin-dev/skills/agent-development/scripts/validate-agent.sh:7-17,51-217`）
- `validate-hook-schema.sh`（hooks 配置校验，`plugins/plugin-dev/commands/create-plugin.md:208,258`；脚本定义 `plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh:30-37,58-159`）
- `test-hook.sh`（hooks 行为测试，`plugins/plugin-dev/commands/create-plugin.md:208-209,259`；脚本定义 `plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh:8-23,183-252`）

### E. 配置、测试、脚本、文档上下文

配置：

- 命令自身前置配置：frontmatter 三元组 + `allowed-tools` 白名单（`plugins/plugin-dev/commands/create-plugin.md:1-5`）。
- 输出配置目标：要求创建 `.claude-plugin/plugin.json`、可选 `.mcp.json`、`hooks/hooks.json`、`.claude/*.local.md` 等（`plugins/plugin-dev/commands/create-plugin.md:116-147,204-225`）。

测试：

- 本目录无独立单测/集成测试代码。
- 依赖流程内“命令式验证 + 用户实测清单”（`plugins/plugin-dev/commands/create-plugin.md:278-304`）。
- 参考测试策略来自 command-development 文档（`plugins/plugin-dev/skills/command-development/references/testing-strategies.md:9-154`）。

脚本：

- 仅“间接依赖” plugin-dev 其他子目录脚本；本目录不包含可执行脚本文件。

文档：

- 上位说明：`plugins/plugin-dev/README.md`。
- 命令规范来源：`plugins/plugin-dev/skills/command-development/SKILL.md` 与 frontmatter reference。

## 关键代码路径与文件引用

### 目标目录

- `plugins/plugin-dev/commands/create-plugin.md`

### 上游调用方/注册入口

- `plugins/plugin-dev/README.md:19-52`（定义命令入口、用法、8 阶段）
- `plugins/README.md:24`（仓库插件总览中声明该命令）
- `.claude-plugin/marketplace.json:106-114`（plugin-dev 在 marketplace 清单中的 source 指向）

### 下游被调用方

- `plugins/plugin-dev/agents/agent-creator.md`
- `plugins/plugin-dev/agents/plugin-validator.md`
- `plugins/plugin-dev/agents/skill-reviewer.md`
- `plugins/plugin-dev/skills/plugin-structure/SKILL.md`
- `plugins/plugin-dev/skills/command-development/SKILL.md`
- `plugins/plugin-dev/skills/agent-development/SKILL.md`
- `plugins/plugin-dev/skills/hook-development/SKILL.md`
- `plugins/plugin-dev/skills/mcp-integration/SKILL.md`
- `plugins/plugin-dev/skills/plugin-settings/SKILL.md`
- `plugins/plugin-dev/skills/skill-development/SKILL.md`

### 关键脚本路径（被命令流程引用）

- `plugins/plugin-dev/skills/agent-development/scripts/validate-agent.sh`
- `plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh`
- `plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh`

## 依赖与外部交互

### 运行时依赖

- Claude Code slash command 机制：把命令 Markdown 作为“给 Claude 的执行指令”加载（`plugins/plugin-dev/skills/command-development/SKILL.md:24-35`）。
- 工具权限依赖：`allowed-tools` 需覆盖流程所需工具，否则流程会被权限中断（`plugins/plugin-dev/commands/create-plugin.md:4`）。
- 插件命名空间与自动发现：命令位于 `commands/` 并在插件上下文下暴露（`plugins/plugin-dev/skills/plugin-structure/SKILL.md:112-115`；`plugins/plugin-dev/skills/command-development/SKILL.md:68-73,576-586`）。

### 与本地 CLI/环境的交互

命令引导用户执行或触发如下外部命令/环境交互：

- `cc --plugin-dir /path/to/plugin-name` 本地加载测试（`plugins/plugin-dev/commands/create-plugin.md:280-282`）。
- hooks 相关脚本依赖 `bash/jq/timeout` 及 `CLAUDE_PLUGIN_ROOT`（`plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh:1-33`；`plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh:170-173,189-191`）。

### 与其他组件的协议交互

- 与 agents 通过 `Task` 工具交互（`plugins/plugin-dev/skills/command-development/references/plugin-features-reference.md:334-356`）。
- 与 skills 通过显式“提及 skill 名称 + Skill tool”交互（`plugins/plugin-dev/commands/create-plugin.md:157-164`；`plugins/plugin-dev/skills/command-development/references/plugin-features-reference.md:360-384`）。
- 与 hooks/mcp/settings 通过“结构约定 + 文件生成 + 校验脚本”交互，不直接实现业务协议逻辑。

## 风险、边界与改进建议

### 风险

1. 阶段计数存在轻微不一致。
- 文档总体定义为 8 阶段（`plugins/plugin-dev/README.md:25`，`plugins/plugin-dev/commands/create-plugin.md:308`）。
- 但 Phase 1 动作里写的是“Create todo list with all 7 phases”（`plugins/plugin-dev/commands/create-plugin.md:29`）。
- 风险：执行代理可能生成 7 项 todo，导致后续状态追踪偏差。

2. 命令质量高度依赖模型遵循度。
- 该目录没有可执行编排器，流程由提示词约束驱动。
- 若模型未严格执行“等待用户确认”节点，可能出现越阶段推进。

3. 测试主要是手工/半自动，不是强约束 CI。
- 本目录无专用自动化测试；验证动作依赖子目录脚本和人工 checklist（`plugins/plugin-dev/commands/create-plugin.md:285-304`）。

4. 工具权限面较大。
- 当前允许 `Read/Write/Grep/Glob/Bash/TodoWrite/AskUserQuestion/Skill/Task`（`plugins/plugin-dev/commands/create-plugin.md:4`）。
- 对新手场景友好，但若执行环境治理严格，权限面可能过宽。

### 边界

1. 本目录只定义“流程命令提示词”，不包含 runtime 代码与协议解析器实现。
2. 不直接维护 plugin manifest、hooks、mcp 的最终正确性，只通过流程要求与下游校验器间接保证。
3. 真实效果取决于运行时是否支持 `Skill` / `Task` / `AskUserQuestion` 等工具。

### 改进建议

1. 修正阶段计数文本一致性。
- 将 `plugins/plugin-dev/commands/create-plugin.md:29` 的 “7 phases” 改为 “8 phases”。

2. 增加“流程完成度检查”子步骤。
- 在 Phase 8 追加一条：核对 TodoWrite 条目数与阶段数一致，避免漏阶段。

3. 补充命令级最小回归脚本或 lint。
- 例如检查 frontmatter 字段完整性、阶段标题存在性、关键工具名（Skill/Task/TodoWrite）是否声明。

4. 收紧工具权限（可选）。
- 若后续可分阶段执行，可考虑在轻量阶段减少 `Bash` 依赖，仅在结构创建/验证阶段启用。

5. 在 README 增加“失败恢复策略”。
- 明确当某阶段校验失败时如何回滚或重试，可降低执行中断后的认知成本。
