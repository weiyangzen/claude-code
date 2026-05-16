# plugins/plugin-dev/skills/hook-development/SKILL.md 研究

## 场景与职责

`plugins/plugin-dev/skills/hook-development/SKILL.md` 是 `plugin-dev` 工具包中负责“Hook 设计与落地”的核心技能规范，定位是把 Claude Code 的事件机制转换成插件内可维护、可测试、可验证的自动化策略。

其职责边界可拆成 4 层：

1. 触发路由层（何时加载）
- 通过 frontmatter `description` 定义明确触发短语与事件关键词（如 PreToolUse/Stop/SessionStart 等），用于让系统在“用户提出 hook 需求”时自动命中该技能。
- 证据：`plugins/plugin-dev/skills/hook-development/SKILL.md:1-5`。

2. 设计决策层（怎么选方案）
- 明确区分 prompt hook（推荐）与 command hook（确定性、外部工具整合），并给出各自适用边界。
- 证据：`plugins/plugin-dev/skills/hook-development/SKILL.md:20-58`。

3. 协议规范层（怎么写配置/输入输出）
- 定义事件配置结构、输入 JSON 字段、输出 JSON 结构、退出码契约、环境变量语义。
- 证据：`plugins/plugin-dev/skills/hook-development/SKILL.md:60-383`。

4. 工程落地层（怎么测试/调试/维护）
- 提供三类配套脚本（schema 校验、脚本测试、脚本 lint），并约束生命周期（会话启动加载、不可热更新、需重启验证）。
- 证据：
  - `plugins/plugin-dev/skills/hook-development/SKILL.md:572-710`
  - `plugins/plugin-dev/skills/hook-development/scripts/README.md:1-164`
  - `plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh:1-159`
  - `plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh:1-252`
  - `plugins/plugin-dev/skills/hook-development/scripts/hook-linter.sh:1-153`

在上下文调用链中：

- 主要调用方（上游）
  - `create-plugin` 工作流在 Phase 5 明确要求 “Hooks: Load hook-development skill”，并在 Phase 6 要求运行 `validate-hook-schema.sh` 与 `test-hook.sh`。
  - 证据：`plugins/plugin-dev/commands/create-plugin.md:157-163,201-209,257-260`。

- 入口文档（产品级导航）
  - `plugin-dev/README.md` 将 Hook Development 定义为七大核心技能之一，并公开其资源构成（3 示例 + 3 参考 + 3 工具脚本）。
  - 证据：`plugins/plugin-dev/README.md:7-17,56-74,271-284`。

- 协同技能（横向依赖）
  - `plugin-structure` 与 `manifest-reference` 负责插件目录/manifest 中 hooks 字段的承载规则。
  - `plugin-validator` 负责在插件验证阶段调用 hook schema 校验思路。
  - `plugin-settings` 给出“设置驱动 Hook 行为”的落地示例。
  - 证据：
    - `plugins/plugin-dev/skills/plugin-structure/SKILL.md:200-231`
    - `plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:259-296,354-367`
    - `plugins/plugin-dev/agents/plugin-validator.md:107-115`
    - `plugins/plugin-dev/skills/plugin-settings/examples/read-settings-hook.sh:1-65`

## 功能点目的

1. 统一 Hook 类型选择策略
- 目标：把“复杂语义判断”与“低延迟确定性检查”分离，降低脚本复杂度。
- 实现：推荐 prompt hook，保留 command hook 处理文件系统/外部工具/性能敏感场景。
- 证据：`plugins/plugin-dev/skills/hook-development/SKILL.md:22-58`。

2. 覆盖完整事件面
- 目标：将 Hook 能力覆盖到工具前后、用户输入、会话生命周期与通知类事件。
- 实现：给出 9 个事件（PreToolUse/PostToolUse/Stop/SubagentStop/SessionStart/SessionEnd/UserPromptSubmit/PreCompact/Notification）及典型用途。
- 证据：`plugins/plugin-dev/skills/hook-development/SKILL.md:121-276,634-644`。

3. 规范输入输出协议，确保可机读
- 目标：避免“脚本能跑但 Claude 不可理解”的情况。
- 实现：定义标准输出字段（continue/suppressOutput/systemMessage）、事件特定输出（permissionDecision/decision）和退出码语义（0/2）。
- 证据：`plugins/plugin-dev/skills/hook-development/SKILL.md:278-299,144-152,202-208`。

4. 确保插件可移植性
- 目标：避免硬编码路径导致插件在不同安装位置失败。
- 实现：强制倡导 `${CLAUDE_PLUGIN_ROOT}`、`${CLAUDE_PROJECT_DIR}`、`${CLAUDE_ENV_FILE}`。
- 证据：`plugins/plugin-dev/skills/hook-development/SKILL.md:322-338,650`。

5. 建立最小测试闭环
- 目标：在进入 Claude 会话前尽早发现错误。
- 实现：先 lint（静态）、再 test-hook（行为）、再 validate-hook-schema（配置），最后 `claude --debug` 端到端验证。
- 证据：
  - `plugins/plugin-dev/skills/hook-development/scripts/README.md:92-128`
  - `plugins/plugin-dev/skills/hook-development/SKILL.md:604-628,707-709`。

6. 支持从“命令脚本校验”迁移到“prompt 校验”
- 目标：提升可维护性与泛化能力，减少硬编码规则。
- 实现：`migration.md` 提供前后对照、迁移 checklist 与混合模式。
- 证据：`plugins/plugin-dev/skills/hook-development/references/migration.md:1-13,55-82,205-243`。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 关键流程

1. 技能触发
- 当用户出现 hook 相关意图词时，依赖 frontmatter 描述触发该技能。
- 入口定义：`plugins/plugin-dev/skills/hook-development/SKILL.md:1-5`。

2. 事件与类型建模
- 先确定事件（例如 PreToolUse/Stop/SessionStart），再确定 hook 类型（prompt/command）。
- 设计指南：`plugins/plugin-dev/skills/hook-development/SKILL.md:121-276`。

3. 配置写入
- 在 `hooks/hooks.json` 中声明 matcher 与 hooks 数组。
- 常用 matcher 语义：精确匹配、多工具 `|`、通配 `*`、正则样式（如 MCP 工具匹配）。
- 证据：`plugins/plugin-dev/skills/hook-development/SKILL.md:340-425`。

4. command hook 脚本实现
- 从 stdin 读取 JSON (`input=$(cat)`)，使用 `jq` 解构字段，输出 JSON + 合规 exit code。
- 示例：
  - 写文件安全校验：`plugins/plugin-dev/skills/hook-development/examples/validate-write.sh:7-38`
  - Bash 安全校验：`plugins/plugin-dev/skills/hook-development/examples/validate-bash.sh:7-43`
  - SessionStart 环境注入：`plugins/plugin-dev/skills/hook-development/examples/load-context.sh:7-55`

5. 离线验证与本地测试
- 配置校验：`validate-hook-schema.sh`
- 行为测试：`test-hook.sh`
- 脚本 lint：`hook-linter.sh`
- 流程建议：`plugins/plugin-dev/skills/hook-development/scripts/README.md:92-128`。

6. 会话级验证
- 重启 Claude 会话加载新 hooks，使用 `/hooks` 与 `claude --debug` 观察注册、执行与 JSON 输入输出。
- 证据：`plugins/plugin-dev/skills/hook-development/SKILL.md:576-599,604-608`。

### 数据结构与协议

1. Hook 配置数据结构（核心）
- 基本结构：`EventName -> [ { matcher, hooks: [hookDef...] } ]`。
- `hookDef` 最小字段：`type`，并且：
  - command 需 `command`
  - prompt 需 `prompt`
  - 可选 `timeout`
- 证据：
  - `plugins/plugin-dev/skills/hook-development/SKILL.md:127-141,344-381`
  - `plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh:65-143`。

2. Hook 输入协议（stdin JSON）
- 通用字段：`session_id/transcript_path/cwd/permission_mode/hook_event_name`
- 事件字段：`tool_name/tool_input/tool_result/user_prompt/reason`
- 证据：`plugins/plugin-dev/skills/hook-development/SKILL.md:300-320`。

3. Hook 输出协议
- 通用输出：`continue/suppressOutput/systemMessage`
- 决策输出：
  - PreToolUse 场景：`hookSpecificOutput.permissionDecision` + 可选 `updatedInput`
  - Stop 场景：`decision=approve|block` + `reason`
- 证据：`plugins/plugin-dev/skills/hook-development/SKILL.md:144-152,202-208,280-293`。

4. 退出码协议
- `0` 成功放行，`2` 阻断反馈给 Claude，其它视为非阻断错误。
- 证据：`plugins/plugin-dev/skills/hook-development/SKILL.md:294-299`。

5. 环境变量协议
- 运行相关：`CLAUDE_PROJECT_DIR`, `CLAUDE_PLUGIN_ROOT`, `CLAUDE_CODE_REMOTE`
- SessionStart 持久化相关：`CLAUDE_ENV_FILE`
- 证据：`plugins/plugin-dev/skills/hook-development/SKILL.md:324-329`。

### 关键命令与脚本逻辑

1. `validate-hook-schema.sh`
- 算法步骤：
  - `jq empty` 校验 JSON 语法。
  - 遍历 root keys 做事件名检查。
  - 遍历每个 event 的 matcher/hooks/type/timeout，统计 error/warning。
- 证据：`plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh:30-158`。

2. `test-hook.sh`
- 功能：
  - 生成样例输入（`--create-sample`）
  - 设置测试环境变量
  - 用 `timeout` 执行脚本并汇总退出码/耗时/输出 JSON 解析
- 证据：`plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh:25-100,170-252`。

3. `hook-linter.sh`
- 检查项覆盖：shebang、安全选项、stdin 读取、jq 使用、变量引用、绝对路径、退出码、stderr 输出、长任务风险等。
- 证据：`plugins/plugin-dev/skills/hook-development/scripts/hook-linter.sh:36-117`。

4. 参考模式库
- `patterns.md`: 常见模式模板（安全、测试门禁、上下文加载、MCP 监控、临时开关、配置驱动）。
- `migration.md`: command→prompt 迁移策略。
- `advanced.md`: 多阶段校验、跨事件状态、外部系统集成、性能优化、故障模式。
- 证据：
  - `plugins/plugin-dev/skills/hook-development/references/patterns.md:5-207,263-343`
  - `plugins/plugin-dev/skills/hook-development/references/migration.md:14-82,155-231`
  - `plugins/plugin-dev/skills/hook-development/references/advanced.md:5-48,85-113,267-377`。

## 关键代码路径与文件引用

### 核心对象（目标文件与同级资产）

1. `plugins/plugin-dev/skills/hook-development/SKILL.md`
- 核心规则、事件说明、协议定义、生命周期约束与实施流程。

2. `plugins/plugin-dev/skills/hook-development/examples/*.sh`
- 三个最小可执行样例：写入校验、Bash 校验、SessionStart 上下文加载。

3. `plugins/plugin-dev/skills/hook-development/scripts/*.sh`
- 三个工具脚本：schema 校验、测试执行、脚本 lint。

4. `plugins/plugin-dev/skills/hook-development/references/*.md`
- 模式库、迁移指导、高级方案。

### 调用方（上游）

1. `plugins/plugin-dev/commands/create-plugin.md`
- 工作流在实现阶段与验证阶段显式调用 hook-development 的知识与工具。
- 关键位点：`plugins/plugin-dev/commands/create-plugin.md:157-163,201-209,257-260`。

2. `plugins/plugin-dev/README.md`
- 公开暴露 hook-development 的触发词、覆盖能力和工具脚本入口。
- 关键位点：`plugins/plugin-dev/README.md:56-74,249-257,271-284`。

3. `plugins/plugin-dev/skills/plugin-structure/README.md`
- 将 hook-development 作为结构规划后的配套技能。
- 关键位点：`plugins/plugin-dev/skills/plugin-structure/README.md:95-100`。

4. `plugins/plugin-dev/skills/skill-development/SKILL.md`
- 将 hook-development 作为“技能编写模板”示例来源。
- 关键位点：`plugins/plugin-dev/skills/skill-development/SKILL.md:608-614`。

### 被调用方（下游）

1. 文档分层下钻
- `references/patterns.md`, `references/migration.md`, `references/advanced.md`。

2. 执行工具链
- `scripts/validate-hook-schema.sh`
- `scripts/test-hook.sh`
- `scripts/hook-linter.sh`

3. 示例脚本
- `examples/validate-write.sh`
- `examples/validate-bash.sh`
- `examples/load-context.sh`

### 配置与验证关联路径

1. Hook 文件位置与 manifest 挂载
- `hooks/hooks.json` 默认路径与 `manifest hooks` 字段路径规则见：
  - `plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:259-290,354-367,423-433`。

2. 插件验证代理的 Hook 校验职责
- `plugins/plugin-dev/agents/plugin-validator.md:107-115`。

## 依赖与外部交互

1. 本地运行时依赖
- Shell 工具链：`bash`, `jq`, `timeout`, `date`, `stat`, `grep`, `sed`, `head`, `tail`, `cat`。
- 证据：三个脚本实现本身（`scripts/*.sh`）。

2. Claude Code 运行语义依赖
- 会话启动加载 hooks、`/hooks` 查看已加载配置、`claude --debug` 调试。
- 证据：`plugins/plugin-dev/skills/hook-development/SKILL.md:576-599,604-608`。

3. 插件系统配置依赖
- hooks 可由 manifest 字段指向文件或内联对象；默认路径 `./hooks/hooks.json`。
- 证据：`plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:259-290,354-361`。

4. 与其他技能/组件的交互
- 与 plugin-structure 协作定义目录与 manifest。
- 与 plugin-validator 协作形成“配置 + 文件”双重校验。
- 与 plugin-settings 协作实现“设置驱动 hook 行为”。
- 证据：
  - `plugins/plugin-dev/skills/plugin-structure/SKILL.md:200-231`
  - `plugins/plugin-dev/agents/plugin-validator.md:107-115`
  - `plugins/plugin-dev/skills/plugin-settings/examples/read-settings-hook.sh:7-65`。

5. 外部系统交互能力（高级场景）
- 文档已覆盖 Slack Webhook、PostgreSQL、StatsD 等外部系统集成模式。
- 证据：`plugins/plugin-dev/skills/hook-development/references/advanced.md:267-311`。

## 风险、边界与改进建议

### 风险 1（高）：Hook 配置格式在文档与工具之间存在冲突

现象：
- `SKILL.md` 前半部分声明“插件 hooks.json 使用 wrapper：`{ "description":..., "hooks": {...} }`”。
  - 证据：`plugins/plugin-dev/skills/hook-development/SKILL.md:64-80,119`。
- 同一文件后半部分示例又改为“root 直接是事件键”。
  - 证据：`plugins/plugin-dev/skills/hook-development/SKILL.md:342-381`。
- `validate-hook-schema.sh` 只接受“root 直接事件键”的结构，遇到 wrapper 会把 `description/hooks` 当未知事件并在后续 `jq` 索引时报错退出。
  - 证据：`plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh:43-67`。
- 实测：wrapper 输入 `WRAPPER_EXIT=5`，direct 输入 `DIRECT_EXIT=0`。

影响：
- 用户严格按前半文档实现会在校验阶段失败，形成“文档-工具互相打架”。

建议：
1. 统一规范口径（推荐与 plugin-structure 一致，统一为 direct 事件结构，manifest 侧由 `hooks` 字段决定挂载方式）。
2. 让 `validate-hook-schema.sh` 同时兼容 wrapper 与 direct，至少给出明确错误信息，不应直接 `jq` 崩溃。
3. 在 `scripts/README.md` 与 `SKILL.md` 增加“被支持格式矩阵”。

### 风险 2（高）：Prompt Hook 支持事件说明存在自相矛盾

现象：
- `SKILL.md` 写明 prompt hook 支持事件是 `Stop/SubagentStop/UserPromptSubmit/PreToolUse`。
  - 证据：`plugins/plugin-dev/skills/hook-development/SKILL.md:34`。
- 但紧接着给出 `PostToolUse` 的 prompt 示例。
  - 证据：`plugins/plugin-dev/skills/hook-development/SKILL.md:155-173`。
- 校验脚本对“非上述 4 事件使用 prompt”只给 warning，不阻断。
  - 证据：`plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh:123-127`。

影响：
- 使用者无法判断 PostToolUse+prompt 是“支持但不推荐”还是“未完全支持”。

建议：
1. 在 `SKILL.md` 增加“support tier”表：`fully supported / experimental / discouraged`。
2. 校验脚本把策略改成可配置（strict 模式下把不支持事件升级为 error）。

### 风险 3（中）：`test-hook.sh` 样例事件覆盖不完整且有字段偏差

现象：
- `Stop|SubagentStop` 分支生成样例时固定写 `hook_event_name: "Stop"`，即使请求的是 `SubagentStop`。
  - 证据：`plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh:59-67`。
- 未提供 `PreCompact` 和 `Notification` 的样例生成。
  - 证据：`plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh:29-97`。

影响：
- 会降低对边缘事件的测试准确性，尤其是子代理场景。

建议：
1. 修正 `SubagentStop` 样例字段。
2. 补充 `PreCompact`/`Notification` 样例模板。

### 风险 4（中）：`load-context.sh` 的 CI 检测条件存在文件/目录判断错误

现象：
- 用 `-f ".github/workflows"` 检测 CI，但该路径通常是目录，应使用 `-d`。
- 证据：`plugins/plugin-dev/skills/hook-development/examples/load-context.sh:50`。

影响：
- 会漏报 GitHub Actions CI 场景，导致 `HAS_CI` 未设置。

建议：
1. 改为 `-d ".github/workflows"`。
2. 为 CI 检测加入单元测试样例（至少 shellcheck + fixture 目录测试）。

### 风险 5（中）：`hook-linter.sh` 的静态检测规则较脆弱

现象：
- 变量引用检测使用正则粗匹配，可能误报/漏报。
  - 证据：`plugins/plugin-dev/skills/hook-development/scripts/hook-linter.sh:67-72`。
- 错误输出检测规则依赖关键字匹配，非英文或自定义消息可能被漏检。
  - 证据：`plugins/plugin-dev/skills/hook-development/scripts/hook-linter.sh:106-111`。

影响：
- 安全问题可能漏掉，或对合法脚本产生噪声告警。

建议：
1. 引入 `shellcheck`（若可用）作为主检测器，当前规则作为补充。
2. 把 lint 输出分为“可信失败”和“启发建议”两级。

### 风险 6（中）：`test-hook.sh` 执行命令拼接存在注入面

现象：
- 通过 `bash -c "cat '$TEST_INPUT' | $HOOK_SCRIPT"` 拼接执行，`HOOK_SCRIPT` 含特殊字符时存在注入风险。
- 证据：`plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh:189-191`。

影响：
- 在复杂路径或恶意输入下，测试工具本身可能执行非预期命令。

建议：
1. 使用数组调用或 `timeout ... bash "$script"` 直接执行，避免 `bash -c` 字符串拼接。
2. 对脚本路径做更严格校验与转义。

### 边界说明

1. 该技能是“规范与模板层”，不是运行时 Hook 引擎实现。
- 它定义最佳实践、样例与工具，不直接实现 Claude Code 内部 Hook 调度器。

2. `plugin-dev` 仓库当前是“工具包内容目录”，不包含本插件自身 `.claude-plugin/plugin.json`。
- 与 `plugin-validator` 里“检查 `.claude-plugin/plugin.json`”的逻辑并不冲突，后者面向“被创建/被验证的目标插件”。
- 证据：`plugins/plugin-dev/agents/plugin-validator.md:51-57`。

3. 参考文档（advanced/patterns）中含大量外部集成示例（curl/psql/nc），属于可选能力，不是默认依赖。

### 建议的优先改进顺序

1. 先统一 hooks 配置格式（文档 + 校验脚本 + README）并补回归测试。
2. 再修复 `test-hook.sh` 样例与命令执行安全问题。
3. 最后提升 lint 可靠性（接入 shellcheck）与 `load-context.sh` 细节正确性。

