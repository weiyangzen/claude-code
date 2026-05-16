# plugins/plugin-dev/skills/plugin-structure/README.md 研究

## 场景与职责

`plugins/plugin-dev/skills/plugin-structure/README.md` 不是运行时协议定义本体，而是 `plugin-structure` 技能目录的“导航与维护入口”。

它的主要职责是把该技能的知识资产做成可维护的目录级索引，帮助维护者与使用者快速回答三类问题：

1. 这个技能覆盖什么问题域（插件目录、manifest、自动发现、命名规范、`${CLAUDE_PLUGIN_ROOT}`）。
2. 这些知识分布在哪些文件（核心 `SKILL.md`、`references/`、`examples/`）。
3. 什么时候触发、与哪些技能协同、后续如何维护。

证据位点：
- 目标能力声明：`plugins/plugin-dev/skills/plugin-structure/README.md:7-13`
- 资源分层定义：`plugins/plugin-dev/skills/plugin-structure/README.md:15-72`
- 触发条件：`plugins/plugin-dev/skills/plugin-structure/README.md:73-84`
- 渐进式加载策略：`plugins/plugin-dev/skills/plugin-structure/README.md:85-93`
- 维护准则：`plugins/plugin-dev/skills/plugin-structure/README.md:102-109`

在 `plugin-dev` 全局中，该 README 的角色是“技能说明书”，而真正执行层是 `/plugin-dev:create-plugin` 与各技能正文：
- `plugin-dev` 总览把 Plugin Structure 列为 7 大技能之一，并在 Quick Start 中引导先用它设计结构：`plugins/plugin-dev/README.md:95-113,214-217,233-235`
- 工作流命令在 Phase 2 强制加载该技能：`plugins/plugin-dev/commands/create-plugin.md:48-52,373-374`

## 功能点目的

### 1) 目录级能力摘要（Overview）

目的：在不打开 `SKILL.md` 的前提下，先确认该技能处理范围，降低误用和错误路由。

实现：README 开头用短列表固定 6 个核心主题（结构、manifest、组件组织、自动发现、路径变量、命名规范）。
- 证据：`plugins/plugin-dev/skills/plugin-structure/README.md:7-13`

### 2) 知识资产编排（Skill Structure）

目的：把长文档拆成“核心规范 + 参考细节 + 实例模板”，平衡上下文体积与可操作性。

实现：明确三层资产并标注每层覆盖面：
- `SKILL.md`：核心概念与工作流
- `references/manifest-reference.md`：manifest 字段、路径与校验
- `references/component-patterns.md`：组件生命周期与组织模式
- `examples/minimal|standard|advanced-plugin.md`：三档模板
- 证据：`plugins/plugin-dev/skills/plugin-structure/README.md:15-72`

### 3) 触发场景声明（When This Skill Triggers）

目的：为技能路由提供高召回的语义入口，减少“本该触发却未触发”。

实现：README 给出用户自然语言触发词集合（create/scaffold/organize/configure 等）。
- 证据：`plugins/plugin-dev/skills/plugin-structure/README.md:75-84`

注：实际运行时触发仍以 `SKILL.md` frontmatter `description` 为准（`plugins/plugin-dev/skills/plugin-structure/SKILL.md:2-4`），README 主要承担“可读化提示”。

### 4) Progressive Disclosure 机制声明

目的：控制上下文开销，避免每次都加载大体量示例。

实现：README 明确“先核心、后引用、再示例”的三级加载顺序，并强调按需加载。
- 证据：`plugins/plugin-dev/skills/plugin-structure/README.md:87-93`
- 与仓库级一致性：`plugins/plugin-dev/README.md:262-269`

### 5) 关联技能与维护策略

目的：让维护者知道横向协作边界与长期更新动作。

实现：
- 关联技能：hook-development、mcp-integration、marketplace-publishing（若可用）
- 维护动作：保持 `SKILL.md` 精简、细节下沉 references、补充 examples、更新版本
- 证据：`plugins/plugin-dev/skills/plugin-structure/README.md:95-109`

## 具体技术实现（关键流程/数据结构/协议/命令）

README 本身是文档索引，不直接定义可执行脚本；其“技术实现”体现在信息架构与运行链路对齐。

### A. 关键流程：从用户请求到资产加载

1. 用户提出插件结构相关请求（例如“set up plugin.json”、“organize components”）。
2. 技能路由命中 `SKILL.md` frontmatter 描述触发词。
   - 触发定义：`plugins/plugin-dev/skills/plugin-structure/SKILL.md:2-4`
3. 先加载 `SKILL.md` 核心规则（目录、字段、发现、命名、排障）。
   - 核心覆盖：`plugins/plugin-dev/skills/plugin-structure/SKILL.md:20-476`
4. 如需深挖，再按 README 指针加载 `references/` 与 `examples/`。
   - README 指针：`plugins/plugin-dev/skills/plugin-structure/README.md:30-72,87-93`
5. 在工作流中，`/plugin-dev:create-plugin` 按阶段把这些规则转成实际目录创建与实现任务。
   - 强制加载：`plugins/plugin-dev/commands/create-plugin.md:48-52`
   - 结构创建：`plugins/plugin-dev/commands/create-plugin.md:116-149`

### B. 结构化数据模型（README 描述的对象模型）

README 描述的资源模型可以抽象为：

- Core：`SKILL.md`
- Reference：`references/*.md`
- Example：`examples/*.md`

并且每层分别承载不同协议深度：
- Core 层定义运行时约束（目录/manifest/发现/命名/排障）
- Reference 层定义字段级语义、解析顺序、组织模式
- Example 层给出可复制模板

对应证据：
- README 结构声明：`plugins/plugin-dev/skills/plugin-structure/README.md:15-72`
- Core 协议：`plugins/plugin-dev/skills/plugin-structure/SKILL.md:20-476`
- Reference 协议：
  - `plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:15-552`
  - `plugins/plugin-dev/skills/plugin-structure/references/component-patterns.md:5-567`

### C. 关联协议与命令（README 间接指向）

README 虽不内置命令，但它指向的正文/参考文档定义了关键协议与命令模式：

1. Manifest 协议（路径、字段、校验）
- 必须路径 `.claude-plugin/plugin.json`：`.../manifest-reference.md:7-10`
- `name` 正则：`.../manifest-reference.md:33-36`
- 路径规则与解析顺序：`.../manifest-reference.md:334-371`

2. 组件发现协议
- 自动发现顺序：`plugins/plugin-dev/skills/plugin-structure/SKILL.md:343-348`

3. 路径变量协议
- `${CLAUDE_PLUGIN_ROOT}` 使用规范：`plugins/plugin-dev/skills/plugin-structure/SKILL.md:258-301`

4. 结构创建命令模板（由 create-plugin 执行）
- `mkdir -p plugin-name/.claude-plugin` 等：`plugins/plugin-dev/commands/create-plugin.md:126-132`

## 关键代码路径与文件引用

### 核心被研究文件

- `plugins/plugin-dev/skills/plugin-structure/README.md`

### 直接上下文（同目录资产）

- `plugins/plugin-dev/skills/plugin-structure/SKILL.md`
- `plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md`
- `plugins/plugin-dev/skills/plugin-structure/references/component-patterns.md`
- `plugins/plugin-dev/skills/plugin-structure/examples/minimal-plugin.md`
- `plugins/plugin-dev/skills/plugin-structure/examples/standard-plugin.md`
- `plugins/plugin-dev/skills/plugin-structure/examples/advanced-plugin.md`

### 调用方（上游）

1. `plugins/plugin-dev/README.md`
- 对外介绍 Plugin Structure 技能能力、资源与使用时机：`95-113`
- Quick Start 直接引导先用该技能：`214-217`

2. `plugins/plugin-dev/commands/create-plugin.md`
- Phase 2 强制加载 plugin-structure：`48-52`
- Phase 4 按结构规则落地目录与 manifest：`116-149`
- Phase 5/6 继续用其它技能补全实现和校验：`157-163,233-260`

3. `plugins/README.md`
- 仓库级插件结构基线与本技能保持一致：`47-61`

### 被调用方（下游）

1. 深度文档：`references/*.md`
2. 模板样例：`examples/*.md`
3. 验收代理：`plugins/plugin-dev/agents/plugin-validator.md:51-134`

## 依赖与外部交互

### 内部依赖

1. 依赖 `plugin-structure/SKILL.md` 作为规范正文；README 只是目录入口。
2. 依赖 `references/` 与 `examples/` 完成“按需下钻”。
3. 依赖 `create-plugin` 将规范转换为具体文件系统操作与配置落地。
4. 依赖 `plugin-validator` 做结构与配置的最终一致性检查。

### 与插件系统交互

1. 通过技能触发机制与 Claude Code 调度交互（触发词来自 `SKILL.md` frontmatter）。
2. 通过 manifest/目录约束与自动发现机制交互。
3. 通过 `${CLAUDE_PLUGIN_ROOT}` 约束与跨安装位置可移植性机制交互。

### 与外部文档/分发体系交互

1. 与仓库 marketplace 清单发生“分发级联”关系：`plugin-dev` 在 marketplace 中注册，进而其技能被可安装使用。
   - 证据：`.claude-plugin/marketplace.json:106-115`
2. 与官方插件文档链接形成外部知识引用：`plugins/README.md:9,45,75-77`

## 风险、边界与改进建议

### 风险与边界

1. README 的字数/规模信息易漂移
- 例如 `SKILL.md (1,619 words)` 这类静态数字会随文档更新失真。
- 证据：`plugins/plugin-dev/skills/plugin-structure/README.md:17,89-91`

2. “Related Skills” 含可用性不确定项
- `marketplace-publishing` 在当前仓库中未见对应技能目录，可能导致误导。
- 证据：`plugins/plugin-dev/skills/plugin-structure/README.md:97-100`

3. README 与执行规范存在“转述层”
- 真实触发与协议在 `SKILL.md`/references；若 README 不同步，会出现认知偏差。

4. README 不包含可执行验证入口
- 与 hook-development 那类带脚本技能相比，缺少“文档一致性自动检查”机制。

### 改进建议

1. 给 README 增加“自动统计生成”字段
- 用脚本自动更新 `word count` 和资源数量，避免手工漂移。

2. 在 Related Skills 标注可用性状态
- 例如“(when available)”进一步明确来源或替代路径，减少错误预期。

3. 新增“与其它技能的规范差异提示”
- README 增加一节列出已知跨技能差异（hooks 配置格式、MCP 文件形态、agent frontmatter 版本），并指向权威来源。

4. 增加一个轻量校验脚本
- 如 `scripts/check-plugin-structure-docs.sh`：检查 README 中列出的 references/examples 是否真实存在、链接是否可达、数字统计是否过期。

5. 在 README 末尾补充“权威优先级”
- 明确冲突时以 `SKILL.md`/`references/manifest-reference.md` 为准，README 为导览层。
