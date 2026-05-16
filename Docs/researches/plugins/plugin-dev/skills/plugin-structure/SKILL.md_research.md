# plugins/plugin-dev/skills/plugin-structure/SKILL.md 研究

## 场景与职责

`plugins/plugin-dev/skills/plugin-structure/SKILL.md` 是 `plugin-dev` 套件中关于“插件骨架与加载契约”的核心规范文件。它直接承担两类职责：

1. 路由职责（何时被加载）
- 通过 frontmatter `description` 定义高覆盖触发短语（create/scaffold/plugin.json/auto-discovery/${CLAUDE_PLUGIN_ROOT} 等），让 Claude 在插件结构问题上命中该技能。
- 证据：`plugins/plugin-dev/skills/plugin-structure/SKILL.md:2-4`

2. 规则职责（加载后给出什么约束）
- 目录结构规范（`.claude-plugin/plugin.json` 必须位置、组件目录根级放置）
- manifest 字段与路径规则
- commands/agents/skills/hooks/MCP 的组织约束
- 自动发现顺序、命名规则、排障路径
- 证据：`plugins/plugin-dev/skills/plugin-structure/SKILL.md:20-476`

在上下游链路中的定位：

- 上游调用方
1. `/plugin-dev:create-plugin` 在 Phase 2 强制加载该技能，并在 Phase 4 按其规范创建目录和 manifest。
   - `plugins/plugin-dev/commands/create-plugin.md:48-52,116-149,373-374`
2. `plugin-dev/README.md` 把它定义为 7 大核心技能之一并给出触发范围。
   - `plugins/plugin-dev/README.md:95-113`

- 下游被调用方
1. `references/manifest-reference.md`：字段级规则、路径解析、校验错误示例。
2. `references/component-patterns.md`：发现/激活生命周期与组织模式。
3. `examples/*.md`：minimal/standard/advanced 三档结构模板。
4. `plugin-validator`：把这些规范转成验收检查点。
   - `plugins/plugin-dev/agents/plugin-validator.md:56-134`

## 功能点目的

### 1) 插件目录标准化

目的：保证插件可被 Claude Code 自动发现，避免“目录存在但组件未注册”。

核心规则：
- `plugin.json` 必须位于 `.claude-plugin/`：`SKILL.md:41,48`
- `commands/agents/skills/hooks` 必须在插件根，不应嵌套到 `.claude-plugin/`：`SKILL.md:42`
- 可选组件按需创建：`SKILL.md:43`

### 2) manifest 最小可用 + 可发布扩展

目的：先满足运行识别，再支持发布元数据与多组件配置。

实现：
- 最小必需 `name`：`SKILL.md:50-63`
- 推荐字段 `version/description/author/homepage/repository/license/keywords`：`SKILL.md:64-84`
- 扩展路径字段 `commands/agents/hooks/mcpServers`：`SKILL.md:86-107`

### 3) 组件组织契约

目的：统一五类组件（commands/agents/skills/hooks/MCP）的落位与格式语义，减少跨团队插件结构分歧。

实现：
- commands：`commands/*.md`，按文件名映射命令语义：`SKILL.md:112-135,307-310`
- agents：`agents/*.md`，frontmatter +角色定义：`SKILL.md:138-163`
- skills：`skills/<name>/SKILL.md`：`SKILL.md:166-199`
- hooks：`hooks/hooks.json` 或 manifest inline：`SKILL.md:202-231`
- MCP：`.mcp.json` 或 manifest inline：`SKILL.md:235-255`

### 4) 路径可移植性

目的：避免安装目录变化导致路径失效。

实现：强制倡导 `${CLAUDE_PLUGIN_ROOT}` 作为插件内部路径锚点，并列出禁止项（硬编码绝对路径、依赖 cwd、`~`）。
- 证据：`SKILL.md:258-301`

### 5) 自动发现与覆盖行为

目的：明确“如何被加载、何时生效、custom path 与默认目录如何并存”。

实现：
- 自动发现顺序（manifest -> commands -> agents -> skills -> hooks -> mcp）：`SKILL.md:343-348`
- custom path 为 supplement，不替换默认目录：`SKILL.md:355`
- 生效时机说明：`SKILL.md:350-353`

### 6) 可维护实践与排障

目的：降低长期维护成本，提供标准排障入口。

实现：
- 组织/命名/可移植/维护四类 best practices：`SKILL.md:357-403`
- 常见故障排查：组件未加载、路径错误、自动发现失效、命名冲突：`SKILL.md:448-472`

## 具体技术实现（关键流程/数据结构/协议/命令）

### A. 关键流程

#### 流程 1：技能触发与规则注入

1. 用户提出插件结构相关请求。
2. 命中 frontmatter description 触发条件。
3. 加载本文件核心规范。
4. 如需细节，下钻 `references/` 与 `examples/`。

证据：
- 触发定义：`SKILL.md:2-4`
- 下钻提示：`SKILL.md:476`

#### 流程 2：create-plugin 工作流中的结构落地

1. Phase 2：强制加载 plugin-structure。
2. Phase 4：执行目录创建与 manifest 初始写入。
3. Phase 5：根据组件类型加载其它技能补全实现。
4. Phase 6：用 validator 与工具脚本做质量检查。

证据：
- `plugins/plugin-dev/commands/create-plugin.md:48-52,116-149,157-163,233-260,373-375`

#### 流程 3：运行时自动发现

1. 读取 `.claude-plugin/plugin.json`。
2. 扫描默认目录（commands/agents/skills/hooks/.mcp.json）。
3. 合并扫描 manifest 自定义路径。
4. 注册组件并在后续场景触发。

证据：
- `SKILL.md:343-355`
- 更细粒度解析顺序：`references/manifest-reference.md:354-371`
- 生命周期模型：`references/component-patterns.md:7-27`

### B. 数据结构与协议

#### 1) Skill frontmatter 协议

```yaml
name: Plugin Structure
description: This skill should be used when ...
version: 0.1.0
```

- `description` 承担触发协议。
- `version` 是技能文档版本，不等于插件版本。
- 证据：`SKILL.md:1-5`

#### 2) 插件目录结构协议

最小结构语法（摘要）：
- 必需：`.claude-plugin/plugin.json`
- 可选：`commands/`, `agents/`, `skills/`, `hooks/`, `.mcp.json`, `scripts/`

证据：`SKILL.md:24-37,39-44`

#### 3) manifest 协议

- 必需字段：`name`（kebab-case）：`SKILL.md:50-63`
- 推荐字段：`version/description/author/...`：`SKILL.md:64-84`
- 路径字段：`commands/agents/hooks/mcpServers`：`SKILL.md:86-107`
- 深入校验与错误示例：`references/manifest-reference.md:15-447`

#### 4) 路径解析协议

- 相对路径、`./` 前缀、禁止绝对路径：`SKILL.md:102-106`
- `${CLAUDE_PLUGIN_ROOT}` 用法与反例：`SKILL.md:258-301`
- 解析顺序与合并规则：`references/manifest-reference.md:354-371`

#### 5) 命名映射协议

- commands 文件名映射 slash 命令名：`code-review.md -> /code-review`
- 证据：`SKILL.md:307-310`

### C. 关键命令与文件系统操作（由工作流执行）

本文件虽不执行命令，但其规范被 create-plugin 直接转译为目录操作：

```bash
mkdir -p plugin-name/.claude-plugin
mkdir -p plugin-name/skills
mkdir -p plugin-name/commands
mkdir -p plugin-name/agents
mkdir -p plugin-name/hooks
```

证据：`plugins/plugin-dev/commands/create-plugin.md:126-132`

### D. 示例实现映射（本技能配套案例）

1. Minimal：最小 manifest + 单命令自动发现。
- `examples/minimal-plugin.md:7-23,64-67`

2. Standard：完整 metadata + commands/agents/skills/hooks 协同。
- `examples/standard-plugin.md:7-35,41-55,426-455`

3. Advanced：多目录 commands/agents + MCP + 分层 hooks + 共享 lib/config。
- `examples/advanced-plugin.md:7-99,131-142,145-175,629-697`

## 关键代码路径与文件引用

### 核心对象（被研究文件）

- `plugins/plugin-dev/skills/plugin-structure/SKILL.md`

### 直接依赖文件（同级）

- `plugins/plugin-dev/skills/plugin-structure/README.md`
- `plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md`
- `plugins/plugin-dev/skills/plugin-structure/references/component-patterns.md`
- `plugins/plugin-dev/skills/plugin-structure/examples/minimal-plugin.md`
- `plugins/plugin-dev/skills/plugin-structure/examples/standard-plugin.md`
- `plugins/plugin-dev/skills/plugin-structure/examples/advanced-plugin.md`

### 调用方（上游）

1. `plugins/plugin-dev/commands/create-plugin.md`
- 强制加载时机：`48-52`
- 结构创建阶段：`116-149`
- 按组件加载其它技能：`157-163`

2. `plugins/plugin-dev/README.md`
- 能力声明与触发短语：`95-113`
- 工作流图中“Design Structure”阶段：`233-235`

3. 仓库插件总览对齐
- `plugins/README.md:47-61`

4. 技能开发元规范引用
- `plugins/plugin-dev/skills/skill-development/SKILL.md:608-615`

### 被调用方（下游）

1. `references/manifest-reference.md`
- 字段与校验细则：`15-552`

2. `references/component-patterns.md`
- 生命周期、组织模式、扩展结构：`5-567`

3. `examples/*.md`
- 三档结构模板（minimal/standard/advanced）

4. `agents/plugin-validator.md`
- 结构验收与安全检查：`51-134`

## 依赖与外部交互

### 内部依赖

1. 依赖 Claude Code 技能触发机制（frontmatter description 语义匹配）。
2. 依赖 plugin 运行时自动发现机制（manifest + 默认目录 + custom paths）。
3. 依赖 `plugin-validator` 做交付前校验。
4. 与 hook-development、mcp-integration、command-development、skill-development 有强耦合的“规范协同关系”。
   - hook-development：`plugins/plugin-dev/skills/hook-development/SKILL.md:62-81`
   - mcp-integration：`plugins/plugin-dev/skills/mcp-integration/SKILL.md:23-37,44-59`
   - command-development：`plugins/plugin-dev/skills/command-development/SKILL.md:576-586,594-599`

### 外部交互

1. 与 marketplace 分发配置交互：`plugin-dev` 被注册后，此技能才能通过插件安装链路分发。
   - `.claude-plugin/marketplace.json:106-115`
2. 与 Claude Code 插件系统标准文档语义对齐。
   - `plugins/README.md:5-10,45-61`

### 测试/脚本依赖边界

1. 本技能目录自身无 `scripts/`，其验证主要依赖外部代理/工具（plugin-validator、hook/agent 各自脚本）。
2. 因此本文件是“规范源”，不是“自动化校验器”。

## 风险、边界与改进建议

### 风险与边界

1. Hooks 配置格式跨技能不一致
- `plugin-structure/SKILL.md` 给出的是事件直接挂根的 hooks 对象（`SKILL.md:215-227`）。
- `hook-development` 明确插件 `hooks/hooks.json` 需 wrapper：`{"hooks": {...}}`（`hook-development/SKILL.md:62-81`）。
- 结果：用户按其中一份示例复制可能触发格式不兼容。

2. Agent frontmatter 示例与 agent-development/validator 口径不一致
- 本文件 agent 示例使用 `description + capabilities`（`SKILL.md:150-157`）。
- validator 期望包含 `name/model/color` 等字段（`agents/plugin-validator.md:91-96`）。
- 结果：按本示例创建 agent 可能在校验阶段告警。

3. `.mcp.json` 形态跨技能存在歧义
- 本文件与 advanced 示例倾向 `{ "mcpServers": { ... } }` 包装（`SKILL.md:239-251`，`examples/advanced-plugin.md:147-175`）。
- `mcp-integration` 示例采用“顶层直接 server map”（`mcp-integration/SKILL.md:27-37`）。
- 结果：用户不确定 `.mcp.json` 标准形态，易出现加载失败或误配置。

4. 自动发现时机描述需要更严谨
- 本文件写“next session 生效”（`SKILL.md:353`），`component-patterns` 强调初始化阶段注册（`component-patterns.md:17`）。
- 建议统一表述为“会话/进程初始化后生效，通常需新会话验证”。

5. 规范多、缺少配套自动 lint
- 本技能缺少结构一致性脚本，规范漂移只能靠人工 review。

### 改进建议

1. 统一 hooks 配置示例
- 在 `plugin-structure` 与 `hook-development` 之间建立单一权威格式，并在另一处加“兼容/迁移说明”。

2. 升级 agent 示例到当前契约
- 将 `SKILL.md` 中 agent 样例改为 `name/description/model/color/tools` 形态，与 validator 一致。

3. 明确 `.mcp.json` 标准形态
- 在 `plugin-structure` 与 `mcp-integration` 两处统一定义，补充“文件形态 vs manifest inline”对照表。

4. 增加本技能专属校验脚本
- 例如 `scripts/validate-plugin-structure.sh`：检查目录层级、manifest 路径与路径字段规范、关键示例一致性。

5. 建立跨技能一致性检查清单
- 在 `plugin-dev/commands/create-plugin.md` 的 Phase 6 增加“cross-skill contract check”，专门检查 hooks/MCP/agent frontmatter 的口径一致。
