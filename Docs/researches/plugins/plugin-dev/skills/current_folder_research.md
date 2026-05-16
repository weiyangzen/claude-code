# plugins/plugin-dev/skills 目录研究（DIR）

## 场景与职责

`plugins/plugin-dev/skills` 是 `plugin-dev` 插件中的“能力规范层 + 工具脚本层”，负责把插件开发知识拆成可触发的 7 个技能，并通过 references/examples/scripts 提供渐进式披露与可执行校验。

目录职责可分为三层：

1. 触发层（何时加载）
- 每个技能通过 `SKILL.md` frontmatter 的 `description` 描述触发语义（例如 hook、MCP、settings、agent 等）。
- 触发入口由 `/plugin-dev:create-plugin` 在不同阶段显式调用 Skill tool：`plugins/plugin-dev/commands/create-plugin.md:157-164`。

2. 规范层（怎么做）
- 7 个 `SKILL.md` 定义插件组件标准：结构、命令、agent、hook、MCP、settings、skill 本身。
- 核心“按需加载”机制由 skill-development 明确：metadata 常驻、SKILL body 按触发加载、references/examples 按需加载：`plugins/plugin-dev/skills/skill-development/SKILL.md:269-277`。

3. 执行层（如何验证）
- 通过 utility scripts 将规范落地为检查/测试：
  - agent: `validate-agent.sh`
  - hooks: `validate-hook-schema.sh` / `test-hook.sh` / `hook-linter.sh`
  - settings: `parse-frontmatter.sh` / `validate-settings.sh`

与上下文依赖关系：

- 调用方（上游）
1. `plugins/plugin-dev/commands/create-plugin.md` 在 Phase 2/5 明确要求加载本目录技能：`48-52`, `157-164`。
2. `plugins/plugin-dev/README.md` 将本目录声明为 plugin-dev 的 7 大核心技能：`7-17`, `54-194`。
3. `plugins/plugin-dev/agents/plugin-validator.md` 在验证插件时会检查 `skills/*/SKILL.md` 并复用本目录脚本理念：`95-109`。

- 被调用方（下游）
1. Claude Code Skill 调度：根据 description 触发各 `SKILL.md`。
2. Claude Code 命令/Agent/Hook/MCP 运行时：消费本目录定义的文件结构、frontmatter、协议与路径约定。
3. Shell 与 CLI 工具链：`bash/sed/awk/grep/jq/timeout`（脚本执行时使用）。

## 功能点目的

### 1) plugin-structure
目的：统一插件目录、manifest、自动发现与路径可移植性。
- 强调 `.claude-plugin/plugin.json` 位置与根目录组件布局：`plugins/plugin-dev/skills/plugin-structure/SKILL.md:15-43`。
- 定义 commands/agents/skills/hooks/.mcp.json 的发现规则：`339-349`。
- 强制 `${CLAUDE_PLUGIN_ROOT}` 作为跨安装位置路径锚点：`258-283`。

### 2) command-development
目的：把 slash command 写法标准化（给 Claude 的指令，不是给用户的说明）。
- 关键原则：commands are instructions FOR Claude：`plugins/plugin-dev/skills/command-development/SKILL.md:30-53`。
- 规范 frontmatter、参数替换、文件引用、bash 嵌入、与 agent/skill/hook 集成：`112-194`, `195-327`, `647-749`。

### 3) agent-development
目的：规范 agent 文件协议、触发示例与 system prompt 结构。
- 定义 required 字段与 `<example>/<commentary>` 触发块：`plugins/plugin-dev/skills/agent-development/SKILL.md:22-114`。
- 定义模型、颜色、工具最小权限策略：`115-160`。
- 提供 AI 辅助生成与人工编写两种流程：`217-259`。

### 4) hook-development
目的：规范事件驱动自动化（PreToolUse/PostToolUse/Stop/...）与安全策略。
- 定义 prompt/command 两类 hooks 及适用事件：`plugins/plugin-dev/skills/hook-development/SKILL.md:20-59`。
- 定义 hook 输入/输出 JSON 与退出码语义：`278-320`。
- 提供安全、性能、调试、生命周期规则：`427-599`。

### 5) mcp-integration
目的：规范 MCP 服务器接入（stdio/SSE/HTTP/ws）、认证与工具命名。
- 配置入口：`.mcp.json`（推荐）或 `plugin.json#mcpServers`：`plugins/plugin-dev/skills/mcp-integration/SKILL.md:23-59`。
- 统一工具命名与命令 pre-allow 策略：`190-223`。
- 生命周期、认证、错误处理与调试路径：`224-239`, `241-476`。

### 6) plugin-settings
目的：建立 `.claude/plugin-name.local.md` 的配置/状态模式。
- YAML frontmatter + markdown body 双区结构：`plugins/plugin-dev/skills/plugin-settings/SKILL.md:11-40`。
- 供 hooks/commands/agents 读取，支持 quick-exit 与默认值：`60-135`, `338-351`。
- 配套解析/校验脚本支持落地：`525-531`。

### 7) skill-development
目的：约束“如何写 skill”，并作为其他 6 个技能的元规范。
- 明确 progressive disclosure 三层加载：`plugins/plugin-dev/skills/skill-development/SKILL.md:77-85`。
- 明确写作风格（description 三人称、body 祈使式）与验证清单：`162-170`, `362-450`。
- 明确插件内 skill 自动发现与本地测试方式：`269-292`。

## 具体技术实现（关键流程/数据结构/协议/命令）

### A. 目录级实现模型：知识 + 参考 + 示例 + 脚本
每个 skill 子目录采用一致结构：
- `SKILL.md`：核心协议与流程
- `references/`：深度资料（本目录 references 总计 11,257 行）
- `examples/`：可复制样例
- `scripts/`：可执行检查/测试（仅部分 skill 有）

这与 skill-development 中的“核心文档瘦身 + 资源按需加载”完全对应：`plugins/plugin-dev/skills/skill-development/SKILL.md:318-360`。

### B. 关键数据结构与协议

1. Skill 协议（frontmatter）
- 基本字段：`name`, `description`, `version`。
- `description` 既是文档元数据，也是触发协议，强调“This skill should be used when ...”。

2. Hook 协议
- 事件集合：PreToolUse/PostToolUse/UserPromptSubmit/Stop/SubagentStop/SessionStart/SessionEnd/PreCompact/Notification：`plugins/plugin-dev/skills/hook-development/SKILL.md:121-277`。
- stdin 输入（通用字段 + 事件字段）：`300-320`。
- stdout 标准结构：`continue/suppressOutput/systemMessage`：`280-293`。
- 退出码：`0`（通过）、`2`（阻断）、其他（非阻断异常）：`294-299`。

3. MCP 协议
- server type: `stdio | sse | http | ws`：`plugins/plugin-dev/skills/mcp-integration/SKILL.md:65-166`。
- 变量展开：`${CLAUDE_PLUGIN_ROOT}` + 用户环境变量：`167-189`。
- 工具命名：`mcp__plugin_<plugin>_<server>__<tool>`：`190-201`。

4. Settings 协议
- 文件：`.claude/<plugin>.local.md`
- frontmatter 解析：`sed` 提取块，`grep+sed` 取字段，`awk` 取正文：`plugins/plugin-dev/skills/plugin-settings/SKILL.md:138-171`。

5. Agent 协议
- required 字段：`name/description/model/color`；`tools` 可选：`plugins/plugin-dev/skills/agent-development/SKILL.md:60-160`。
- description 需携带 `<example>` 块，作为触发训练数据：`82-114`。

6. Command 协议（被本目录文档消费并反向约束）
- frontmatter 关键字段：`description/allowed-tools/model/argument-hint/disable-model-invocation`：`plugins/plugin-dev/skills/command-development/SKILL.md:112-194`。
- 动态参数与文件引用：`195-315`。

### C. 关键流程（从调用到验证）

1. create-plugin 编排调用技能
- Phase 2 强制先 load `plugin-structure`：`plugins/plugin-dev/commands/create-plugin.md:48-52`。
- Phase 5 按组件类型分配 skill：`157-164`。
- Phase 6 指定脚本化验证：`255-260`。

2. script 化验证链
- Agent: `bash plugins/plugin-dev/skills/agent-development/scripts/validate-agent.sh <agent.md>`
- Hooks:
  - `bash plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh <hooks.json>`
  - `bash plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh [opts] <script> <input.json>`
  - `bash plugins/plugin-dev/skills/hook-development/scripts/hook-linter.sh <script...>`
- Settings:
  - `bash plugins/plugin-dev/skills/plugin-settings/scripts/parse-frontmatter.sh <file> [field]`
  - `bash plugins/plugin-dev/skills/plugin-settings/scripts/validate-settings.sh <file>`

3. 测试/调试通道
- hooks 与 MCP 均建议用 `claude --debug`：`plugins/plugin-dev/skills/hook-development/SKILL.md:602-609`, `plugins/plugin-dev/skills/mcp-integration/SKILL.md:445-455`。
- MCP 通过 `/mcp` 校验 server/tool 可见性：`plugins/plugin-dev/skills/mcp-integration/SKILL.md:238-239`, `425-431`。

## 关键代码路径与文件引用

### 核心入口与调用方
- `plugins/plugin-dev/commands/create-plugin.md:48-52`
- `plugins/plugin-dev/commands/create-plugin.md:157-164`
- `plugins/plugin-dev/README.md:54-194`
- `plugins/plugin-dev/agents/plugin-validator.md:95-109`

### 技能主文件
- `plugins/plugin-dev/skills/plugin-structure/SKILL.md`
- `plugins/plugin-dev/skills/command-development/SKILL.md`
- `plugins/plugin-dev/skills/agent-development/SKILL.md`
- `plugins/plugin-dev/skills/hook-development/SKILL.md`
- `plugins/plugin-dev/skills/mcp-integration/SKILL.md`
- `plugins/plugin-dev/skills/plugin-settings/SKILL.md`
- `plugins/plugin-dev/skills/skill-development/SKILL.md`

### 脚本与执行链
- `plugins/plugin-dev/skills/agent-development/scripts/validate-agent.sh`
- `plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh`
- `plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh`
- `plugins/plugin-dev/skills/hook-development/scripts/hook-linter.sh`
- `plugins/plugin-dev/skills/plugin-settings/scripts/parse-frontmatter.sh`
- `plugins/plugin-dev/skills/plugin-settings/scripts/validate-settings.sh`
- `plugins/plugin-dev/skills/hook-development/scripts/README.md`

### 参考与样例（被调用文档）
- plugin-structure: `references/manifest-reference.md`, `references/component-patterns.md`, `examples/*.md`
- command-development: `references/*.md`（frontmatter/interactive/testing/plugin-features 等）
- agent-development: `references/system-prompt-design.md`, `references/triggering-examples.md`, `examples/*.md`
- hook-development: `references/patterns.md`, `references/migration.md`, `references/advanced.md`, `examples/*.sh`
- mcp-integration: `references/server-types.md`, `references/authentication.md`, `references/tool-usage.md`, `examples/*.json`
- plugin-settings: `references/parsing-techniques.md`, `references/real-world-examples.md`, `examples/*`
- skill-development: `references/skill-creator-original.md`

### 配置与协议相关路径
- `.claude-plugin/plugin.json`（规范定义位置）：`plugins/plugin-dev/skills/plugin-structure/SKILL.md:15-49`
- `hooks/hooks.json`（hook 配置入口）
- `.mcp.json`（MCP 配置入口）
- `.claude/*.local.md`（settings/state）

## 依赖与外部交互

### 本地命令依赖
- 必需：`bash`, `sed`, `awk`, `grep`, `head`, `tail`
- 关键附加：
  - `jq`（hooks/settings JSON 解析）
  - `timeout`（hook 测试超时）

证据：
- `validate-hook-schema.sh` 直接执行 `jq empty`：`plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh:30-35`
- `test-hook.sh` 使用 `jq` 校验与 `timeout` 执行：`plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh:155-158`, `189-191`

### 环境变量与运行时接口
- Hook 脚本运行时变量：`CLAUDE_PROJECT_DIR`, `CLAUDE_PLUGIN_ROOT`, `CLAUDE_ENV_FILE`, `CLAUDE_CODE_REMOTE`：`plugins/plugin-dev/skills/hook-development/SKILL.md:324-330`。
- MCP 与 command 的路径可移植接口：`${CLAUDE_PLUGIN_ROOT}`：
  - plugin-structure: `258-283`
  - command-development: `529-573`
  - mcp-integration: `167-176`

### 与外部系统交互
- MCP 服务器交互（stdio 进程 / SSE / HTTP / ws）
- OAuth/Token 认证流程（由 mcp-integration references 细化）
- 文档外链（Claude Docs / MCP Docs）作为规范来源：
  - `plugins/plugin-dev/skills/hook-development/SKILL.md:693`
  - `plugins/plugin-dev/skills/mcp-integration/SKILL.md:535-537`

### 测试与验证依赖
- 本目录没有统一 CI 测试套件；主要依赖脚本和手工验证流程。
- command-development 的 `references/testing-strategies.md` 提供测试框架建议，但并未在本目录形成自动化 harness。

## 风险、边界与改进建议

### 风险 1（高）：Hook 配置格式与校验脚本不一致

现状：
- hook-development 在“Plugin hooks.json Format”声明插件格式应为 wrapper：`{"description":..., "hooks": {...}}`：`plugins/plugin-dev/skills/hook-development/SKILL.md:64-80`。
- 同一文件后文“Plugin Hook Configuration”又给出顶层事件格式（无 wrapper）：`342-381`。
- `validate-hook-schema.sh` 仅按顶层事件键校验：`plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh:41-43`。

实测：
- 对 wrapper 输入运行脚本会出现 `Cannot index string with number` 并异常退出（exit=5）。

建议：
1. 校验器显式兼容两种格式（wrapper/direct），先归一化成 event map 再校验。
2. `hook-development/SKILL.md` 保留单一规范并在示例区统一。

### 风险 2（高）：`set -e` + `((count++))` 导致校验脚本提前退出

现状：
- `validate-agent.sh` 与 `validate-hook-schema.sh` 使用 `set -euo pipefail` + `((error_count++))`/`((warning_count++))`。
- 在 bash 下 `((count++))` 当旧值为 `0` 时返回状态码 `1`，会触发 `set -e` 提前退出。

实测：
- `validate-agent.sh` 在首个 warning 后即退出，无法输出完整汇总（例如校验 `plugins/plugin-dev/agents/plugin-validator.md`）。
- `validate-hook-schema.sh` 在首个错误后即退出，无法累计错误列表。

建议：
1. 改为 `((++warning_count))` / `((++error_count))`，或 `warning_count=$((warning_count+1))`。
2. 增加最小回归测试，覆盖“有 warning 仍应继续检查到汇总”的行为。

### 风险 3（中）：agent-development 文档引用不存在脚本

现状：
- `agent-development/SKILL.md` 列出 `test-agent-trigger.sh`：`plugins/plugin-dev/skills/agent-development/SKILL.md:398-400`。
- 实际 `plugins/plugin-dev/skills/agent-development/scripts/` 仅有 `validate-agent.sh`。

建议：
1. 要么补齐 `test-agent-trigger.sh`。
2. 要么删除该引用并给出替代测试步骤。

### 风险 4（中）：跨技能规范漂移（manifest/agent 格式）

现状：
1. plugin-structure 要求 manifest 位于 `.claude-plugin/plugin.json`：`plugins/plugin-dev/skills/plugin-structure/SKILL.md:15-49`。
2. command-development 的插件树示例却写 `plugin-name/plugin.json`（根目录）：`plugins/plugin-dev/skills/command-development/SKILL.md:579-586`。
3. plugin-structure 的示例 agent 仍采用旧式字段（`description/capabilities`），而 agent-development 要求 `name/model/color`：
   - 示例：`plugins/plugin-dev/skills/plugin-structure/examples/standard-plugin.md:132-141`
   - 规范：`plugins/plugin-dev/skills/agent-development/SKILL.md:60-143`

建议：
1. 统一示例到当前协议版本（manifest 路径 + agent frontmatter）。
2. 在各 skill 顶部增加“协议版本/兼容声明”。

### 风险 5（低）：示例脚本细节错误可能误导用户

现状：
- `load-context.sh` 用 `-f` 检查 `.github/workflows`（通常是目录），条件可能误判：`plugins/plugin-dev/skills/hook-development/examples/load-context.sh:50`。

建议：
1. 改为 `-d .github/workflows`。
2. 把 examples 与 scripts 分级标注（示例/生产可用）。

### 边界说明

1. 本目录以“规范文档 + 参考资料 + 脚本工具”为主，不是业务功能实现；其质量依赖运行时调度行为与文档一致性。
2. 大量能力通过 examples/references 间接提供，若跨文件约定漂移，用户会在实操阶段才暴露问题。
3. 测试手段以脚本和人工验证为主，缺少统一自动化回归是当前主要工程边界。

### 总体改进优先级

1. 先修脚本可用性（风险 1/2）。
2. 再修文档一致性（风险 3/4）。
3. 最后补齐自动化回归与示例质量（风险 5 + 测试边界）。
