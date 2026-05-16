# plugins/plugin-dev 目录研究（DIR）

## 场景与职责

### 目录定位
`plugins/plugin-dev` 是一个“插件开发插件”（meta-plugin）：它不直接提供某个业务域能力，而是提供开发 Claude Code 插件的方法论、模板、校验工具和引导流程。

- 在插件总览中被声明为“7 个技能 + 3 个 agent + 1 个 8 阶段命令工作流”：`plugins/README.md:24`
- 入口文档将其定义为「Plugin Development Toolkit」并强调渐进式披露（progressive disclosure）：`plugins/plugin-dev/README.md:1-17`
- 核心入口命令是 `/plugin-dev:create-plugin`：`plugins/plugin-dev/README.md:21-53`

### 组件构成（当前仓库）
- 顶层文档/入口：`README.md`、`commands/create-plugin.md`
- Agent：`agents/agent-creator.md`、`agents/plugin-validator.md`、`agents/skill-reviewer.md`
- Skill：7 个主题（agent-development / command-development / hook-development / mcp-integration / plugin-settings / plugin-structure / skill-development）
- 支撑资料：`references/`、`examples/`
- 可执行脚本：6 个（agent/hook/settings 三类）

对应目录结构见：
- 目录列表：`plugins/plugin-dev/*`
- 标准插件结构参考（上游约定）：`plugins/README.md:47-61`

### 与上下文依赖的关系
- 调用方（上游）
1. Claude Code 的命令系统加载 `commands/create-plugin.md` 并暴露 `/plugin-dev:create-plugin`：`plugins/plugin-dev/commands/create-plugin.md:1-5`
2. Claude Code 的技能调度系统按描述触发 `skills/*/SKILL.md`：`plugins/plugin-dev/skills/plugin-structure/SKILL.md:339-348`、`plugins/plugin-dev/skills/skill-development/SKILL.md:269-277`
3. Claude Code 的 agent 调度系统按 `agents/*.md` frontmatter + 示例触发 agent：`plugins/plugin-dev/agents/agent-creator.md:1-35`

- 被调用方（下游）
1. Skill 工具：在工作流中按阶段加载对应技能：`plugins/plugin-dev/commands/create-plugin.md:48-52`、`157-164`
2. Task/Agent：调用 `agent-creator`、`plugin-validator`、`skill-reviewer`：`plugins/plugin-dev/commands/create-plugin.md:15-16`、`194-200`、`238-251`
3. Bash + jq + timeout 等 CLI：由 `validate-agent.sh`、`validate-hook-schema.sh`、`test-hook.sh` 等脚本执行：
   - `plugins/plugin-dev/skills/agent-development/scripts/validate-agent.sh:1-217`
   - `plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh:1-159`
   - `plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh:1-252`

## 功能点目的

### 1) 一站式插件创建编排
`/plugin-dev:create-plugin` 把“从需求到可发布插件”拆成 8 阶段：Discovery → Planning → Design → Structure → Implementation → Validation → Testing → Documentation：`plugins/plugin-dev/commands/create-plugin.md:24-341`。

目的：
- 防止直接跳到实现，要求先澄清需求与组件边界：`13-19`、`83-99`
- 在每个阶段强制加载对应 skill，复用规范：`48-52`、`157-164`、`371-375`
- 把质量门禁前置到 phase 6（validator + 脚本）：`233-266`

### 2) 七大能力知识库（skills）
每个 skill 提供：
- 触发语义（frontmatter description）
- 核心流程（SKILL.md）
- 深入参考（references）
- 可运行例子/脚本（examples / scripts）

总览见：`plugins/plugin-dev/README.md:54-194`。

### 3) 三个专业 agent
- `agent-creator`：把用户需求转为 agent frontmatter + system prompt：`plugins/plugin-dev/agents/agent-creator.md:41-123`
- `plugin-validator`：对 manifest、目录、commands/agents/skills/hooks/mcp 做完整检查：`plugins/plugin-dev/agents/plugin-validator.md:41-142`
- `skill-reviewer`：重点审查 skill 描述触发质量与渐进式披露设计：`plugins/plugin-dev/agents/skill-reviewer.md:60-105`

### 4) 脚本化验证与测试
- Agent 结构校验：`validate-agent.sh`
- Hook 配置校验 + Hook 行为测试 + Hook linter：`validate-hook-schema.sh`、`test-hook.sh`、`hook-linter.sh`
- 设置文件解析与结构校验：`parse-frontmatter.sh`、`validate-settings.sh`

定位：把纯提示词规范转成“可执行检查”，降低人工 review 负担。

## 具体技术实现（关键流程/数据结构/协议/命令）

### A. 主工作流命令的编排协议
`commands/create-plugin.md` 不是普通文档，而是给 Claude 的“流程执行 DSL”。

关键 frontmatter：
- `description` / `argument-hint`
- `allowed-tools` 明确允许 `Skill`、`Task`、`TodoWrite`、`AskUserQuestion`：`plugins/plugin-dev/commands/create-plugin.md:1-5`

关键流程机制：
1. 每阶段输出明确目标与产物（Output）
2. 多处“必须等待用户确认”的 gating points：`363-369`
3. phase 5 明确组件到 skill 的映射：`157-164`
4. phase 6 明确校验动作与提问策略：`238-268`

### B. Skill 渐进式披露机制（Progressive Disclosure）
统一三层模型：
1. metadata 常驻
2. SKILL.md 按触发加载
3. references/examples/scripts 按需加载

见：`plugins/plugin-dev/README.md:262-269`，`plugins/plugin-dev/skills/skill-development/SKILL.md:77-85`。

这直接影响上下文占用与命中率，是 plugin-dev 的核心设计理念。

### C. Hook 协议与测试工具链
#### Hook 输入/输出与退出码
- stdin 输入 JSON（含 `hook_event_name/session_id/cwd/...`）：`plugins/plugin-dev/skills/hook-development/SKILL.md:300-320`
- 标准输出结构：`continue/suppressOutput/systemMessage`：`278-293`
- 退出码约定：`0` 成功，`2` 阻断，其它异常：`294-299`

#### Hook 测试脚本技术点
`test-hook.sh`：
- 支持 `--create-sample <event>` 自动生成测试输入：`25-100`
- 注入 `CLAUDE_PROJECT_DIR` / `CLAUDE_PLUGIN_ROOT` / `CLAUDE_ENV_FILE`：`170-179`
- 用 `timeout` 包裹执行并解析退出码语义：`189-218`
- 自动尝试 JSON 解析输出：`226-230`

`validate-hook-schema.sh`：
- `jq empty` 做 JSON 语法门禁：`30-36`
- 枚举事件、校验 matcher/hooks/type/timeout：`41-143`

### D. Agent 文件协议与校验
#### Agent 文件结构协议
- YAML frontmatter：`name/description/model/color/tools`
- description 要求 `<example>` 与 `<commentary>` 触发样例
- 正文为二人称 system prompt

见：`plugins/plugin-dev/skills/agent-development/SKILL.md:22-58`、`82-114`、`162-197`。

#### validate-agent.sh 检查逻辑
- frontmatter 起止检查：`33-45`
- 字段检查与格式约束：`58-168`
- prompt 长度和结构提示：`170-203`
- 错误/告警汇总并据此 exit：`208-217`

### E. 插件设置（.local.md）解析模式
`plugin-settings` 采用 `.claude/plugin-name.local.md`（YAML frontmatter + markdown body）：`plugins/plugin-dev/skills/plugin-settings/SKILL.md:11-19`、`22-40`。

关键解析手法：
- `sed` 抽 frontmatter：`138-143`
- `grep+sed` 抽字段：`145-162`
- `awk` 抽正文：`164-171`

对应工具：
- `parse-frontmatter.sh`：按字段提取：`plugins/plugin-dev/skills/plugin-settings/scripts/parse-frontmatter.sh:37-58`
- `validate-settings.sh`：校验 marker/frontmatter/body/常见布尔字段：`plugins/plugin-dev/skills/plugin-settings/scripts/validate-settings.sh:40-101`

### F. MCP 集成协议
核心配置位：`.mcp.json` 或 `plugin.json#mcpServers`：`plugins/plugin-dev/skills/mcp-integration/SKILL.md:23-59`。

协议要点：
- server type：`stdio/sse/http/ws`：`65-166`
- 环境变量展开：`${CLAUDE_PLUGIN_ROOT}` + 用户 env：`167-189`
- 工具命名：`mcp__plugin_<plugin>_<server>__<tool>`：`190-223`
- 生命周期：插件加载 → 配置解析 → 连接/进程启动 → 工具注册：`224-237`

### G. 命令开发协议
- 命令是“写给 Claude 的执行指令，不是写给用户”：`plugins/plugin-dev/skills/command-development/SKILL.md:30-53`
- frontmatter 字段：`description/allowed-tools/model/argument-hint/disable-model-invocation`：`112-194`
- 动态参数：`$ARGUMENTS` 和 `$1/$2...`：`195-263`
- 与插件组件集成（Task 调 agent、触发 skill、配合 hook）：`647-749`

## 关键代码路径与文件引用

### 顶层入口
- `plugins/plugin-dev/README.md`
- `plugins/plugin-dev/commands/create-plugin.md`
- `plugins/README.md:24`（plugin-dev 在插件总览中的注册）

### Agent 路径
- `plugins/plugin-dev/agents/agent-creator.md`
- `plugins/plugin-dev/agents/plugin-validator.md`
- `plugins/plugin-dev/agents/skill-reviewer.md`

### Skill 主入口
- `plugins/plugin-dev/skills/plugin-structure/SKILL.md`
- `plugins/plugin-dev/skills/command-development/SKILL.md`
- `plugins/plugin-dev/skills/agent-development/SKILL.md`
- `plugins/plugin-dev/skills/hook-development/SKILL.md`
- `plugins/plugin-dev/skills/mcp-integration/SKILL.md`
- `plugins/plugin-dev/skills/plugin-settings/SKILL.md`
- `plugins/plugin-dev/skills/skill-development/SKILL.md`

### 关键参考文档
- Manifest/path/discovery：`plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:259-360`
- 组件生命周期与组织模式：`plugins/plugin-dev/skills/plugin-structure/references/component-patterns.md:5-27`
- 命令字段规范：`plugins/plugin-dev/skills/command-development/references/frontmatter-reference.md:5-130`
- 命令与 agent/skill/hook 集成：`plugins/plugin-dev/skills/command-development/references/plugin-features-reference.md:330-407`

### 可执行脚本
- `plugins/plugin-dev/skills/agent-development/scripts/validate-agent.sh`
- `plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh`
- `plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh`
- `plugins/plugin-dev/skills/hook-development/scripts/hook-linter.sh`
- `plugins/plugin-dev/skills/plugin-settings/scripts/parse-frontmatter.sh`
- `plugins/plugin-dev/skills/plugin-settings/scripts/validate-settings.sh`

### 示例与配置样例
- Hook 示例：`plugins/plugin-dev/skills/hook-development/examples/*.sh`
- MCP 示例：`plugins/plugin-dev/skills/mcp-integration/examples/*.json`
- Settings 示例：`plugins/plugin-dev/skills/plugin-settings/examples/*`

## 依赖与外部交互

### 本地依赖（CLI/运行时）
- Shell 工具链：`bash`, `sed`, `awk`, `grep`, `head`, `tail`, `timeout`, `date`
- JSON 工具：`jq`（hook schema 校验、hook 输入解析）

证据：
- `validate-hook-schema.sh` 强依赖 `jq`：`plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh:32-33`
- `test-hook.sh` 依赖 `jq` + `timeout`：`plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh:155-156`、`190`

### 与 Claude Code 运行时的交互
- 通过 command frontmatter 协议声明工具权限：`plugins/plugin-dev/commands/create-plugin.md:4`
- 通过环境变量与 Hook/脚本交互：`CLAUDE_PROJECT_DIR`、`CLAUDE_PLUGIN_ROOT`、`CLAUDE_ENV_FILE`：`plugins/plugin-dev/skills/hook-development/SKILL.md:324-329`
- 通过 `/mcp` 观察 MCP server 注册：`plugins/plugin-dev/skills/mcp-integration/SKILL.md:238-239`

### 与仓库其它插件的耦合（文档/模式层）
`plugin-dev` 的规范会被其它插件“反向引用”作为规范源（尤其 hook、manifest、skill discoverability）。例如：
- `plugins/security-guidance/hooks/hooks.json:1-16`
- `plugins/hookify/hooks/hooks.json:1-49`

这些插件文件的结构是否与 `plugin-dev` 提供的校验器兼容，直接影响 toolchain 可用性。

## 风险、边界与改进建议

### 已识别风险
1. **Hook 配置格式与校验脚本存在不一致**
- skill 文档声明 plugin hooks.json 采用 wrapper：`{"description":..., "hooks": {...}}`：`plugins/plugin-dev/skills/hook-development/SKILL.md:64-80`
- 仓库内多个插件确实使用 wrapper：`plugins/security-guidance/hooks/hooks.json:1-4`、`plugins/hookify/hooks/hooks.json:1-4`
- 但 `validate-hook-schema.sh` 按“顶层即事件键”校验：`plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh:41-43`
- 实测对 wrapper 文件会报错并异常退出（`Cannot index string with number`）

2. **agent-development 文档引用了不存在的脚本**
- 文档声明有 `test-agent-trigger.sh`：`plugins/plugin-dev/skills/agent-development/SKILL.md:399`
- 实际 `scripts/` 仅存在 `validate-agent.sh`

3. **plugin-validator 文档尾部存在与当前主题无关残留句**
- `plugins/plugin-dev/agents/plugin-validator.md:184` 为疑似编辑残留（提到 “all 6 skills are documented...”），与 validator 系统提示体不一致。

4. **plugin-dev 目录自身缺少 `.claude-plugin/plugin.json`**
- 仓库其它插件普遍具备该文件（结构规范要求也如此）：`plugins/README.md:47-61`
- 当前 `plugins/plugin-dev` 下未见该 manifest（会限制按“标准插件目录”直接启用的一致性）

### 边界条件
1. 本目录多数内容是“规范与提示词资产”，不是应用业务代码；正确性高度依赖 Claude 运行时解释与触发策略。
2. 测试能力主要依赖脚本 smoke test；目录内没有系统化单元/集成测试框架（仅见 `test-hook.sh` 这类工具脚本）。
3. 很多示例依赖外部环境（MCP 服务、环境变量、用户授权），离线无法端到端验证。

### 改进建议（按优先级）
1. **统一 Hook schema 规范与校验器实现**
- 让 `validate-hook-schema.sh` 同时支持：
  - plugin wrapper 格式 `{"hooks": {...}}`
  - settings direct 格式 `{ "PreToolUse": [...] }`
- 并对 `description` 字段做显式忽略。

2. **修复文档-脚本漂移**
- 补齐 `test-agent-trigger.sh`，或删除 `SKILL.md` 的无效引用并改成可执行验证步骤。

3. **清理 agent 文本残留并做 lint**
- 对 `agents/*.md` 增加最小 lint（frontmatter 完整性 + 无关尾句检测），避免把会话残留写入系统提示。

4. **补全 plugin-dev 自身 manifest**
- 增加 `plugins/plugin-dev/.claude-plugin/plugin.json`，与仓库其它插件保持一致，便于本地 `--plugin-dir` 直接加载验证。

5. **增加基础回归脚本**
- 至少覆盖：
  - `hooks/hooks.json` 格式兼容性回归
  - `agents/*.md` 结构校验回归
  - `skills/*/SKILL.md` 引用文件存在性检查
