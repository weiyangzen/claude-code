# plugins/plugin-dev/skills/plugin-structure 目录研究（DIR）

## 场景与职责

`plugins/plugin-dev/skills/plugin-structure` 是 `plugin-dev` 工具包中负责“插件骨架规范”的技能目录，核心职责是把 Claude Code 插件从“能跑”提升到“可发现、可维护、可扩展”的结构化形态。

它在整体体系中的定位可拆成三层：

1. 路由职责（何时触发）
- `SKILL.md` 的 description 明确触发语义：当用户提到创建插件、搭建目录、配置 `plugin.json`、启用自动发现、使用 `${CLAUDE_PLUGIN_ROOT}` 时触发该技能。
- 证据：`plugins/plugin-dev/skills/plugin-structure/SKILL.md:2-4`。

2. 规范职责（如何组织）
- 定义插件目录层级、`plugin.json` 字段、组件（commands/agents/skills/hooks/MCP）组织、命名规范、路径规范与自动发现策略。
- 证据：`plugins/plugin-dev/skills/plugin-structure/SKILL.md:20-355`，`references/manifest-reference.md:11-552`，`references/component-patterns.md:5-567`。

3. 教学职责（如何落地）
- 提供 `minimal/standard/advanced` 三类样例，覆盖从最小可用到企业级组织模式。
- 证据：`plugins/plugin-dev/skills/plugin-structure/README.md:50-72`，`examples/*.md`。

在上下游依赖中，该目录并非“代码 import 关系”，而是“工作流编排 + 运行时规则约束”关系：

- 上游调用方（谁要求使用它）
1. `/plugin-dev:create-plugin` 在 Phase 2 明确“必须加载 plugin-structure skill”，并在 Phase 4 按该技能规则创建目录和 manifest。
  证据：`plugins/plugin-dev/commands/create-plugin.md:48-52`，`116-149`，`371-374`。
2. `plugin-dev` 总览把它定义为第 3 个核心技能，指定触发词与资源结构。
  证据：`plugins/plugin-dev/README.md:95-113`。

- 下游被调用方（它引导加载谁）
1. `SKILL.md` 末尾明确引导读取 `references/` 与 `examples/` 深化细节。
  证据：`plugins/plugin-dev/skills/plugin-structure/SKILL.md:474-476`。
2. `plugin-validator` 在校验阶段会检查本技能定义的结构约束（manifest、目录、组件格式、路径可移植性）。
  证据：`plugins/plugin-dev/agents/plugin-validator.md:51-123`。

## 功能点目的

该目录主要功能点及其目标如下：

1. 插件目录标准化
- 目标：统一插件根目录布局，避免“组件放错层级导致无法发现”。
- 关键规则：`plugin.json` 必须在 `.claude-plugin/`；`commands/agents/skills/hooks` 必须位于插件根。
- 证据：`plugins/plugin-dev/skills/plugin-structure/SKILL.md:24-44`。

2. manifest 最小必需与扩展元数据
- 目标：先保证最小可识别（`name`），再支持发布所需元信息（`version/description/author/repository/license/keywords`）。
- 证据：`plugins/plugin-dev/skills/plugin-structure/SKILL.md:50-84`，`references/manifest-reference.md:15-208`。

3. 组件路径配置与默认目录合并
- 目标：在保留默认自动发现的前提下支持分层组织与多目录扩展。
- 关键行为：自定义路径是 supplement，不替换默认目录。
- 证据：`plugins/plugin-dev/skills/plugin-structure/SKILL.md:86-107`，`339-355`，`references/manifest-reference.md:211-371`。

4. 组件格式协议统一
- 目标：确保 commands/agents/skills/hooks/MCP 都能按 Claude Code 约定被解析。
- 证据：`plugins/plugin-dev/skills/plugin-structure/SKILL.md:108-255`。

5. 路径可移植性约束
- 目标：消除安装路径差异带来的脚本失效，统一使用 `${CLAUDE_PLUGIN_ROOT}`。
- 证据：`plugins/plugin-dev/skills/plugin-structure/SKILL.md:256-301`。

6. 自动发现机制说明
- 目标：让开发者理解“何时被扫描、何时生效、如何排查未加载”。
- 证据：`plugins/plugin-dev/skills/plugin-structure/SKILL.md:339-467`，`references/component-patterns.md:7-27`。

7. 分层样例驱动
- 目标：根据插件复杂度提供可直接套用的结构模板：
  - 最小模式：单命令快速起步；
  - 标准模式：命令 + agent + skill + hooks；
  - 高级模式：多目录命令/agent、MCP、共享库、环境配置、分层 hooks。
- 证据：`plugins/plugin-dev/skills/plugin-structure/examples/minimal-plugin.md:5-83`，`standard-plugin.md:5-587`，`advanced-plugin.md:5-765`。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 渐进式披露实现模型

目录采用典型三层知识资产组织：

1. `SKILL.md`：核心规范与主流程（轻量高频知识）。
2. `references/`：manifest 与组织模式深度说明（低频但高细节）。
3. `examples/`：可复用的整包示例（落地模板）。

证据：`plugins/plugin-dev/skills/plugin-structure/README.md:15-93`。

这让运行时可先靠触发词激活核心说明，再按问题深度加载 reference/example，避免一次性塞入所有上下文。

### 2) 核心数据结构与协议

#### A. Skill frontmatter 协议

```yaml
name: Plugin Structure
description: This skill should be used when ...
version: 0.1.0
```

- `description` 承担触发协议作用；
- `version` 仅用于技能文档版本，不等同于插件 manifest 版本。
- 证据：`plugins/plugin-dev/skills/plugin-structure/SKILL.md:1-5`。

#### B. 插件 manifest 协议（`.claude-plugin/plugin.json`）

最小结构：

```json
{ "name": "plugin-name" }
```

扩展结构覆盖 metadata 与组件路径字段（`commands/agents/hooks/mcpServers`）。
- `name` 校验正则：`/^[a-z][a-z0-9]*(-[a-z0-9]+)*$/`；
- 路径字段支持 string 或 string[]（不同字段略有差异）；
- `hooks`/`mcpServers` 支持“路径”或“内联对象”双形态。
- 证据：`plugins/plugin-dev/skills/plugin-structure/SKILL.md:46-107`，`references/manifest-reference.md:15-330`。

#### C. 组件协议

1. commands：`commands/*.md` + YAML frontmatter。
2. agents：`agents/*.md` + YAML frontmatter。
3. skills：`skills/<skill>/SKILL.md`。
4. hooks：`hooks/hooks.json` 或 manifest inline。
5. MCP：`.mcp.json` 或 manifest inline。

证据：`plugins/plugin-dev/skills/plugin-structure/SKILL.md:110-255`。

#### D. 路径协议

1. manifest 路径需相对插件根，且以 `./` 开头；
2. 禁止绝对路径、`../`、反斜杠路径；
3. 运行时引用脚本/资源统一 `${CLAUDE_PLUGIN_ROOT}`。

证据：`plugins/plugin-dev/skills/plugin-structure/SKILL.md:260-301`，`references/manifest-reference.md:334-350`。

### 3) 关键流程：从编排到生效

#### 流程 A：create-plugin 编排链路（调用方流程）

1. Phase 2：先加载 `plugin-structure`，决定需要哪些组件。
2. Phase 4：创建 `.claude-plugin/` 与组件目录，写入 `plugin.json`。
3. Phase 5：再按组件类型加载其他技能进行实现。

关键命令片段（文档中直接给出）：

```bash
mkdir -p plugin-name/.claude-plugin
mkdir -p plugin-name/skills
mkdir -p plugin-name/commands
mkdir -p plugin-name/agents
mkdir -p plugin-name/hooks
```

证据：`plugins/plugin-dev/commands/create-plugin.md:48-52`，`116-149`，`157-164`，`371-374`。

#### 流程 B：自动发现与激活（运行时流程）

1. 读取 `.claude-plugin/plugin.json`；
2. 扫描默认目录；
3. 合并扫描 manifest 自定义路径；
4. 注册组件；
5. 初始化 hooks/MCP；
6. 在使用阶段按类型激活（命令调用、agent 选择、skill 匹配、hook 事件触发、MCP 工具转发）。

证据：`plugins/plugin-dev/skills/plugin-structure/SKILL.md:341-355`，`references/component-patterns.md:7-27`，`references/manifest-reference.md:352-371`。

### 4) 样例中的技术实现要点

#### minimal 模式
- 只有 `.claude-plugin/plugin.json` + `commands/hello.md`；
- 演示“最小 manifest + 自动发现单命令”。
- 证据：`plugins/plugin-dev/skills/plugin-structure/examples/minimal-plugin.md:7-67`。

#### standard 模式
- 引入 commands + agents + skills + hooks + scripts；
- hooks 演示 prompt 与 command 两类 hook 并行使用；
- 命令脚本调用统一走 `${CLAUDE_PLUGIN_ROOT}`。
- 证据：`plugins/plugin-dev/skills/plugin-structure/examples/standard-plugin.md:8-35`，`57-93`，`426-455`，`457-504`。

#### advanced 模式
- 通过 manifest 自定义多目录 `commands`/`agents`；
- `.mcp.json` 中挂载多 MCP server；
- hooks 脚本按安全/质量/流程分层；
- 引入 `lib/` 与 `config/` 强化跨组件复用。
- 证据：`plugins/plugin-dev/skills/plugin-structure/examples/advanced-plugin.md:11-99`，`131-142`，`145-175`，`629-697`，`714-765`。

### 5) 验证与测试路径（上下文依赖）

目标目录自身没有 `scripts/` 或测试文件，校验主要依赖外部：

1. `plugin-validator`：校验目录结构、manifest、组件格式、hooks/MCP 配置。
  证据：`plugins/plugin-dev/agents/plugin-validator.md:51-123`。
2. create-plugin Phase 6：调用其他技能里的验证脚本（如 `validate-hook-schema.sh`、`validate-agent.sh` 等）。
  证据：`plugins/plugin-dev/commands/create-plugin.md:153-163`，`255-260`。

## 关键代码路径与文件引用

### 目标目录（被研究对象）

- `plugins/plugin-dev/skills/plugin-structure/SKILL.md`
- `plugins/plugin-dev/skills/plugin-structure/README.md`
- `plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md`
- `plugins/plugin-dev/skills/plugin-structure/references/component-patterns.md`
- `plugins/plugin-dev/skills/plugin-structure/examples/minimal-plugin.md`
- `plugins/plugin-dev/skills/plugin-structure/examples/standard-plugin.md`
- `plugins/plugin-dev/skills/plugin-structure/examples/advanced-plugin.md`

### 调用方（上游）

1. `plugins/plugin-dev/commands/create-plugin.md:48-52`
- Phase 2 强制加载 `plugin-structure`。

2. `plugins/plugin-dev/commands/create-plugin.md:116-149`
- Phase 4 直接按该技能规范创建目录与 `plugin.json`。

3. `plugins/plugin-dev/README.md:95-113`
- 将 `plugin-structure` 声明为核心技能并定义触发场景。

4. `plugins/plugin-dev/README.md:214-217`
- Quick Start 第一步直接引导用户使用该技能设计结构。

### 被调用方/耦合对象（下游与横向）

1. `plugins/plugin-dev/agents/plugin-validator.md:51-123`
- 把本技能规则转换为实际验收清单。

2. `plugins/plugin-dev/skills/skill-development/SKILL.md:614`
- 将 `../plugin-structure/` 作为“组织良好”的技能样例。

3. `plugins/README.md:47-61`
- 仓库级插件结构说明与本技能规则一致，形成跨目录共识。

4. `plugins/plugin-dev/skills/command-development/references/plugin-features-reference.md:16-40`
- 命令发现与命名空间规则与本技能的结构约束互补。

### 关键配置路径

- `.claude-plugin/plugin.json`：插件元数据与组件路径入口。
- `hooks/hooks.json`：事件钩子配置。
- `.mcp.json`：MCP server 配置（或由 manifest 内联）。
- `commands/`、`agents/`、`skills/`：默认自动发现目录。

## 依赖与外部交互

### 本地依赖与执行环境

1. 该目录本身为文档资产，不包含可执行脚本或测试程序。
2. 实际执行依赖 Claude Code 运行时的插件发现、hooks、MCP、agent/skill 调度。
3. 验证环节依赖 `plugin-validator` 与其他技能目录脚本（`bash/jq` 工具链）。

### 与外部系统/协议的交互

1. Hook 协议：通过 `hooks/hooks.json` 声明事件处理并执行命令或 prompt hook。
  证据：`plugins/plugin-dev/skills/plugin-structure/SKILL.md:202-231`。
2. MCP 协议：通过 `.mcp.json`/`mcpServers` 配置外部工具服务。
  证据：`plugins/plugin-dev/skills/plugin-structure/SKILL.md:233-255`。
3. 路径可移植交互：`${CLAUDE_PLUGIN_ROOT}` 作为跨安装位置的稳定锚点。
  证据：`plugins/plugin-dev/skills/plugin-structure/SKILL.md:258-301`。

### 文档/知识依赖

1. 仓库级插件文档：`plugins/README.md`。
2. plugin-dev 总览：`plugins/plugin-dev/README.md`。
3. 横向技能（agent-development/hook-development/mcp-integration/command-development）提供格式与运行细节补充。

## 风险、边界与改进建议

### 风险 1（高）：跨技能规范存在格式漂移（agents）

现象：
- `plugin-structure` 的示例 agent frontmatter 使用 `description + capabilities` 风格；
- `agent-development` 将 `name/description/model/color` 定义为 required。

证据：
- `plugins/plugin-dev/skills/plugin-structure/examples/standard-plugin.md:133-141`
- `plugins/plugin-dev/skills/agent-development/SKILL.md:60-131`

影响：
- 用户按 `plugin-structure` 样例创建 agent 后，可能在 `plugin-validator` 或运行时触发格式不一致问题。

建议：
1. 将 `plugin-structure` 示例 agent frontmatter 升级到 agent-development 当前规范。
2. 在示例里标注“教学简化版/旧格式”或增加迁移提示。

### 风险 2（中）：`.mcp.json` 结构与兄弟技能文档不一致

现象：
- `plugin-structure` advanced 示例使用 `{ "mcpServers": { ... } }`；
- `mcp-integration` 推荐示例多为“顶层直接 server map”。

证据：
- `plugins/plugin-dev/skills/plugin-structure/examples/advanced-plugin.md:147-175`
- `plugins/plugin-dev/skills/mcp-integration/SKILL.md:23-37`
- `plugins/plugin-dev/skills/mcp-integration/examples/stdio-server.json:1-26`

影响：
- 开发者在两个技能间切换时可能不确定 canonical schema。

建议：
1. 在两个技能中统一声明 `.mcp.json` 推荐结构与兼容形态。
2. 给出“可直接 copy/paste 的单一标准样例”。

### 风险 3（中）：manifest 路径约束在横向文档中有表达差异

现象：
- plugin-structure 明确 `plugin.json` 位于 `.claude-plugin/`；
- command-development 的结构示意图显示根目录 `plugin.json`。

证据：
- `plugins/plugin-dev/skills/plugin-structure/SKILL.md:41-48`
- `plugins/plugin-dev/skills/command-development/SKILL.md:579-586`

影响：
- 新用户可能按错误路径建 manifest，导致插件无法被识别。

建议：
1. 统一所有 skill 的目录示意图到 `.claude-plugin/plugin.json`。
2. 在 `plugin-validator` 报错文案中补充“常见误放路径”提示。

### 风险 4（中）：自动发现生效时机描述存在歧义

现象：
- 同一节同时出现“no restart required”和“changes take effect on next Claude Code session”。

证据：
- `plugins/plugin-dev/skills/plugin-structure/SKILL.md:350-354`

影响：
- 用户难以判断是否需要重开会话/重启进程来看到新增组件。

建议：
1. 明确区分“插件启用时生效”与“当前会话热加载能力”。
2. 增加一个简短验证步骤（例如重开 session 后 `/help` 检查命令可见性）。

### 风险 5（中）：目标目录缺少可执行校验脚本，文档回归能力弱

现象：
- 目录内仅文档和示例，没有结构 lint 或样例一致性检查脚本。

影响：
- 样例与横向规范漂移时不易被及时发现。

建议：
1. 新增轻量脚本（如 `scripts/validate-plugin-structure-docs.sh`）检查：
   - 示例中的 manifest 路径；
   - agent frontmatter 与当前 agent-development 规范一致性；
   - `${CLAUDE_PLUGIN_ROOT}` 使用覆盖率。
2. 在 CI 或日常文档维护中执行该脚本。

### 风险 6（低）：Related Skills 中包含当前仓库不存在的技能名

现象：
- `README.md` 列出 `marketplace-publishing`，但 `plugins/plugin-dev/skills/` 下无对应目录。

证据：
- `plugins/plugin-dev/skills/plugin-structure/README.md:95-101`

影响：
- 用户按文档追踪时会遇到跳转断点。

建议：
1. 标注 “when available” 的同时给出替代路径，或
2. 暂时移除该项，待技能落地后再恢复。

### 边界说明

1. 本目录不直接实现插件运行逻辑，仅定义“结构规范 + 样例模板”。
2. 真正的加载/触发/执行行为由 Claude Code 运行时与其他技能（hook/mcp/agent/command）协作完成。
3. 因此该目录质量主要受“文档一致性、跨技能协议对齐、样例可执行性”影响。
