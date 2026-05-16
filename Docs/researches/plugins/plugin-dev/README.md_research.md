# `plugins/plugin-dev/README.md` 研究文档

## 场景与职责

### 1. 目标对象定位
`plugins/plugin-dev/README.md` 是 `plugin-dev` 插件的总入口说明文档，职责不是运行时代码执行，而是定义该插件的能力边界、推荐工作流与资源导航。

- 文档声明该插件是“插件开发工具包”，覆盖 hooks / MCP / structure / settings / commands / agents / skills 七大方向：`plugins/plugin-dev/README.md:1-17`
- 文档把 `/plugin-dev:create-plugin` 作为一体化入口，定义 8 阶段开发流程：`plugins/plugin-dev/README.md:21-53`
- 在仓库级插件目录索引中，`plugin-dev` 被作为“7 skills + 3 agents + 1 command”的开发类插件展示：`plugins/README.md:24`
- 在 marketplace 元数据中，`plugin-dev` 由 `.claude-plugin/marketplace.json` 发布登记：` .claude-plugin/marketplace.json:106-115`

### 2. 上下游依赖（调用方 / 被调用方）

调用方（上游）：

1. 插件目录总览文档（`plugins/README.md`）把该 README 作为插件说明入口。
2. marketplace 清单（`.claude-plugin/marketplace.json`）把 `./plugins/plugin-dev` 暴露给 `/plugin install` 流程。
3. 用户通过 `/plugin install plugin-dev@claude-code-marketplace` 或 `cc --plugin-dir /path/to/plugin-dev` 进入该插件文档路径：`plugins/plugin-dev/README.md:196-208`

被调用方（下游）：

1. 命令入口：`plugins/plugin-dev/commands/create-plugin.md`（8 阶段执行协议）
2. Agents：`agent-creator.md`、`plugin-validator.md`、`skill-reviewer.md`
3. Skills：7 个 `skills/*/SKILL.md`
4. 脚本：hook / settings / agent 三类校验与测试脚本（共 6 个）
5. 参考资料与示例：`references/`、`examples/`（为渐进式披露提供下钻材料）

### 3. 与仓库上下文的关系

`README.md` 自身不直接被“执行”，但它通过文档约定影响三类实际执行对象：

1. Claude Code 命令系统执行 `commands/create-plugin.md`（frontmatter +流程文本）。
2. Claude Code agent 调度执行 `agents/*.md`（frontmatter + system prompt）。
3. Claude Code skill 调度按描述触发 `skills/*/SKILL.md`，并按需读取 references/examples/scripts。

---

## 功能点目的

### 1. 统一插件创建流程

README 把插件开发拆分为 Discovery/Planning/Design/Structure/Implementation/Validation/Testing/Documentation 八阶段，目的是：

1. 避免直接编码，先澄清需求与边界。
2. 在实现阶段按组件类型加载对应 skill，减少随意实现。
3. 在验证阶段引入 validator agent 和脚本化检查，形成质量门禁。

对应执行规范见：`plugins/plugin-dev/commands/create-plugin.md:24-341`

### 2. 七类能力模块化

README 对每个 skill 给出“触发语句 + 覆盖能力 + 资源”，目的是把插件开发关键能力标准化：

1. Hook Development：事件驱动自动化与策略拦截
2. MCP Integration：外部工具/服务接入
3. Plugin Structure：目录与 manifest 规范
4. Plugin Settings：`.claude/*.local.md` 配置模式
5. Command Development：斜杠命令协议
6. Agent Development：agent frontmatter 与 prompt 设计
7. Skill Development：技能的渐进式披露与触发质量

见：`plugins/plugin-dev/README.md:54-194`

### 3. 渐进式披露（Progressive Disclosure）

README 明确三层加载模型（metadata -> SKILL.md -> references/examples），目标是平衡上下文开销与知识深度：`plugins/plugin-dev/README.md:262-269`。该模型在 `skill-development` 中有同源定义：`plugins/plugin-dev/skills/skill-development/SKILL.md:77-85`。

### 4. 脚本化质量保障

README 把 utility scripts 作为“可执行的质量工具链”，目的是把规范落地为可验证检查：

- Hook schema 校验：`validate-hook-schema.sh`
- Hook 行为测试：`test-hook.sh`
- Hook 脚本 lint：`hook-linter.sh`
- Agent 文件校验：`validate-agent.sh`
- Settings 解析/校验：`parse-frontmatter.sh`、`validate-settings.sh`

脚本路径见：
- `plugins/plugin-dev/skills/hook-development/scripts/README.md:1-164`
- `plugins/plugin-dev/skills/agent-development/scripts/validate-agent.sh:1-217`
- `plugins/plugin-dev/skills/plugin-settings/scripts/*.sh`

---

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1. `/plugin-dev:create-plugin` 执行协议

该命令是完整流程编排器，frontmatter 允许工具集合包括 `Skill`、`Task`、`TodoWrite`、`AskUserQuestion`：`plugins/plugin-dev/commands/create-plugin.md:1-5`。

关键流程点：

1. Phase 2 强制加载 `plugin-structure`：`plugins/plugin-dev/commands/create-plugin.md:48-52`
2. Phase 5 按组件类型强制加载对应 skills：`plugins/plugin-dev/commands/create-plugin.md:157-164`
3. Phase 6 要求调用 `plugin-validator` 并按结果修复：`plugins/plugin-dev/commands/create-plugin.md:233-268`
4. 全流程设置 “Wait for User” 决策门：`plugins/plugin-dev/commands/create-plugin.md:363-369`

### 2. 组件发现与装载协议

`plugin-structure` skill 给出 Claude Code 自动发现顺序：

1. 读取 `.claude-plugin/plugin.json`
2. 扫描 `commands/*.md`
3. 扫描 `agents/*.md`
4. 扫描 `skills/*/SKILL.md`
5. 加载 `hooks/hooks.json` 或 manifest hooks
6. 加载 `.mcp.json` 或 manifest mcp

见：`plugins/plugin-dev/skills/plugin-structure/SKILL.md:339-348`

并且 custom path 是“补充而非替代”默认路径：`plugins/plugin-dev/skills/plugin-structure/SKILL.md:355`、`plugins/plugin-dev/skills/plugin-structure/references/component-patterns.md:11-17`

### 3. 核心数据结构与文件协议

#### 3.1 命令（Command）协议

- 载体：Markdown + YAML frontmatter
- 常见字段：`description`、`argument-hint`、`allowed-tools`
- 本质是“写给 Claude 的执行指令”，不是写给用户的说明文本

见：`plugins/plugin-dev/skills/command-development/SKILL.md:30-53`、`98-120`

#### 3.2 Agent 协议

- frontmatter：`name`、`description`、`model`、`color`、`tools`
- `description` 要求 `<example>` 场景块提升触发可靠性
- 正文为系统提示词（职责/流程/输出格式）

见：`plugins/plugin-dev/agents/agent-creator.md:75-123`、`plugins/plugin-dev/skills/agent-development/scripts/validate-agent.sh:51-203`

#### 3.3 Hook 协议

- 事件集合：PreToolUse/PostToolUse/Stop/SubagentStop/SessionStart/SessionEnd/UserPromptSubmit/PreCompact/Notification
- 输入：stdin JSON（含 `session_id`、`cwd`、`hook_event_name` 等）
- 输出：可返回 `hookSpecificOutput` / `systemMessage`
- 退出码：`0` 成功，`2` 阻断

见：`plugins/plugin-dev/skills/hook-development/SKILL.md:121-320`

#### 3.4 MCP 协议

- 配置位置：`.mcp.json` 或 `plugin.json#mcpServers`
- server type：`stdio`、`sse`、`http`、`ws`
- 工具命名：`mcp__plugin_<plugin>_<server>__<tool>`

见：`plugins/plugin-dev/skills/mcp-integration/SKILL.md:19-237`

#### 3.5 Settings 协议

- 文件：`.claude/plugin-name.local.md`
- 结构：YAML frontmatter + markdown body
- 解析惯用法：`sed` 抽 frontmatter、`grep+sed` 抽字段、`awk` 抽正文

见：`plugins/plugin-dev/skills/plugin-settings/SKILL.md:11-19`、`136-171`

### 4. 关键命令与脚本流程

1. Hook 配置校验：`./validate-hook-schema.sh hooks/hooks.json`
2. Hook 行为测试：`./test-hook.sh my-hook.sh test-input.json`
3. Hook 脚本 lint：`./hook-linter.sh my-hook.sh`
4. Agent 校验：`./validate-agent.sh path/to/agent.md`
5. Settings 校验：`./validate-settings.sh .claude/my-plugin.local.md`
6. Frontmatter 提取：`./parse-frontmatter.sh .claude/my-plugin.local.md enabled`

脚本说明见：`plugins/plugin-dev/skills/hook-development/scripts/README.md:5-123`

---

## 关键代码路径与文件引用

### 1. 目标文件与主入口

1. `plugins/plugin-dev/README.md`（当前研究对象）
2. `plugins/plugin-dev/commands/create-plugin.md`（核心执行入口）
3. `plugins/README.md:24`（仓库级注册说明）
4. `.claude-plugin/marketplace.json:106-115`（marketplace 元数据注册）

### 2. 直接依赖（被 README 引导访问）

1. `plugins/plugin-dev/agents/agent-creator.md`
2. `plugins/plugin-dev/agents/plugin-validator.md`
3. `plugins/plugin-dev/agents/skill-reviewer.md`
4. `plugins/plugin-dev/skills/*/SKILL.md`（7 个）

### 3. 关键参考文档（协议来源）

1. `plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:7-10`
2. `plugins/plugin-dev/skills/plugin-structure/references/component-patterns.md:11-27`
3. `plugins/plugin-dev/skills/command-development/references/frontmatter-reference.md:60-129`
4. `plugins/plugin-dev/skills/hook-development/scripts/README.md:5-164`

### 4. 测试/校验脚本路径

1. `plugins/plugin-dev/skills/agent-development/scripts/validate-agent.sh`
2. `plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh`
3. `plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh`
4. `plugins/plugin-dev/skills/hook-development/scripts/hook-linter.sh`
5. `plugins/plugin-dev/skills/plugin-settings/scripts/parse-frontmatter.sh`
6. `plugins/plugin-dev/skills/plugin-settings/scripts/validate-settings.sh`

---

## 依赖与外部交互

### 1. 本地运行时依赖

1. Shell 工具链：`bash`、`sed`、`awk`、`grep`、`head`、`tail`、`timeout`
2. JSON 解析：`jq`（hook validator/test 脚本强依赖）

证据：
- `plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh:30-33`
- `plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh:155-191`

### 2. Claude Code 运行时交互

1. 工具权限依赖 frontmatter `allowed-tools` 声明：`plugins/plugin-dev/commands/create-plugin.md:4`
2. hooks 脚本依赖环境变量：`CLAUDE_PROJECT_DIR`、`CLAUDE_PLUGIN_ROOT`、`CLAUDE_ENV_FILE`：`plugins/plugin-dev/skills/hook-development/SKILL.md:324-329`
3. MCP 服务器生命周期由插件启用阶段触发：`plugins/plugin-dev/skills/mcp-integration/SKILL.md:226-237`

### 3. 外部协议与服务

1. MCP 对接外部服务时使用 `sse/http/ws` 网络协议：`plugins/plugin-dev/skills/mcp-integration/SKILL.md:95-166`
2. HTTP/SSE 场景涉及 OAuth 或 Token 认证约定：`plugins/plugin-dev/skills/mcp-integration/SKILL.md:243-270`

### 4. 文档与发布链路交互

1. README 安装说明依赖 marketplace 名称 `plugin-dev@claude-code-marketplace`：`plugins/plugin-dev/README.md:198-202`
2. 插件在仓库的发布入口是 `.claude-plugin/marketplace.json`：`.claude-plugin/marketplace.json:106-115`

---

## 风险、边界与改进建议

### 1. 风险

1. 统计信息漂移风险（README 数字与仓库现状不一致）
   - README 声称核心技能约 `~11,065` 词：`plugins/plugin-dev/README.md:306`
   - 实测 7 个 `SKILL.md` 合计约 `14,075` 词（`wc -w` 本地统计）
   - README 声称 command-development 核心 `1,535` 词：`plugins/plugin-dev/README.md:148`
   - 实测 `plugins/plugin-dev/skills/command-development/SKILL.md` 为 `2,426` 词

2. Hook 配置格式存在规范冲突
   - `hook-development` 文档前部要求 plugin hooks.json 使用 wrapper（`{"hooks": {...}}`）：`plugins/plugin-dev/skills/hook-development/SKILL.md:62-80`
   - 同一文档后部又给出顶层事件直出示例：`plugins/plugin-dev/skills/hook-development/SKILL.md:340-357`
   - `validate-hook-schema.sh` 只按“顶层键=事件名”遍历：`plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh:41-43`、`65-66`

3. 命令 frontmatter 规范冲突
   - `plugin-validator` 写明 `allowed-tools` 应为数组：`plugins/plugin-dev/agents/plugin-validator.md:82`
   - `frontmatter-reference` 明确允许字符串或数组：`plugins/plugin-dev/skills/command-development/references/frontmatter-reference.md:62-86`

4. `plugin-validator` 文本尾部疑似残留
   - 文件末尾存在与 validator 角色无关的会话性语句：`plugins/plugin-dev/agents/plugin-validator.md:184`

5. 安装说明与仓库目录形态存在张力
   - README 建议 `cc --plugin-dir /path/to/plugin-dev`：`plugins/plugin-dev/README.md:204-208`
   - 但本仓库 `plugins/plugin-dev` 目录当前没有 `.claude-plugin/plugin.json`（而结构文档强调该文件必需）：`plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:7-10`

### 2. 边界

1. 该对象主要是规范文档，不是业务逻辑代码；正确性依赖运行时解释与工具链一致性。
2. 插件目录内没有统一 CI 测试入口，更多依赖脚本工具和人工场景验证。
3. MCP 与 OAuth 场景依赖外部服务与凭据，离线无法做完整端到端验证。

### 3. 改进建议（按优先级）

1. 统一 hooks 配置协议
   - 明确单一规范（wrapper 或 direct）或让脚本同时支持两种格式。
   - 在 `validate-hook-schema.sh` 中先判断是否存在 `.hooks` 包装层。

2. 修复校验规则冲突
   - 对齐 `plugin-validator` 与 `frontmatter-reference` 的 `allowed-tools` 类型定义。

3. 建立 README 自动体检
   - 用脚本自动计算 skill/refs/examples/utils 数量和词数，防止统计陈旧。

4. 清理 agent 文档残留文本
   - 为 `agents/*.md` 增加 lint（frontmatter 完整性 + 非法尾注检测）。

5. 明确 plugin-dev 在仓库中的“发布态 vs 开发态”加载方式
   - 若要支持 `--plugin-dir` 直载，应补齐 `.claude-plugin/plugin.json`；
   - 若仓库内仅作 marketplace 源数据，应在 README 显式说明该前提。

