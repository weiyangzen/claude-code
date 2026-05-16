# plugins/feature-dev/.claude-plugin 目录研究（DIR）

## 场景与职责

`plugins/feature-dev/.claude-plugin` 是 `feature-dev` 插件的 manifest 目录，当前仅包含一个文件：`plugin.json`（`plugins/feature-dev/.claude-plugin/plugin.json:1-9`）。

在 Claude Code 插件体系中，该目录承担“插件发现入口与身份声明”职责，而不承载具体业务流程执行：
- 插件 manifest 必须位于 `.claude-plugin/plugin.json`，否则不会被识别（`plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:7-10`）。
- 组件生命周期的第一步是读取已启用插件的 manifest（`plugins/plugin-dev/skills/plugin-structure/references/component-patterns.md:11-16`）。
- 仓库级 marketplace 将 `feature-dev` 的 `source` 指向插件根 `./plugins/feature-dev`，随后运行时进入该插件读取本目录 manifest（`.claude-plugin/marketplace.json:62-70`）。

因此，本目录是“插件被装配”的锚点：没有它，`/feature-dev` 命令及其 3 个 agent 的后续自动发现都不会发生。

## 功能点目的

### 1. 声明插件身份与元信息
`plugin.json` 定义了 `name/version/description/author`：
- `name: feature-dev`
- `version: 1.0.0`
- `description: Comprehensive feature development workflow ...`
- `author: Sid Bidasaria`
见 `plugins/feature-dev/.claude-plugin/plugin.json:2-8`。

这些字段用于插件识别、冲突检测、展示与发布元数据，其中 `name` 是核心唯一标识（`plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:15-31`）。

### 2. 触发后续组件自动发现
`feature-dev` 的执行能力分布在 `commands/` 与 `agents/`：
- `/feature-dev` 命令定义：`plugins/feature-dev/commands/feature-dev.md:1-125`
- `code-explorer` / `code-architect` / `code-reviewer`：`plugins/feature-dev/agents/*.md`

由于 manifest 未声明自定义 `commands` 或 `agents` 路径，运行时依赖默认目录自动发现（`plugins/plugin-dev/skills/plugin-structure/SKILL.md:110-141,343-347,355`）。manifest 的存在是这一默认发现链路的前提。

### 3. 为 marketplace 与文档体系提供可对齐元数据
- marketplace 对应条目复写了 `feature-dev` 的 name/version/description/author/source（`.claude-plugin/marketplace.json:62-70`）。
- 插件总览文档把它声明为“7-phase workflow + 3 agents”（`plugins/README.md:20`）。

这使安装展示层（marketplace）与能力说明层（README）可以围绕同一插件标识协同工作。

## 具体技术实现（关键流程/数据结构/协议/命令）

### A. 关键流程（调用方 -> 目标 -> 被调用方）

1. 调用方（上游注册与发现）
- `.claude-plugin/marketplace.json` 注册 `feature-dev`，并以 `source: ./plugins/feature-dev` 指向插件根（`.claude-plugin/marketplace.json:62-70`）。
- Claude Code 组件发现阶段读取 `.claude-plugin/plugin.json`（`plugins/plugin-dev/skills/plugin-structure/references/component-patterns.md:11-16`）。

2. 目标对象（本目录）
- `plugins/feature-dev/.claude-plugin/plugin.json:1-9`

3. 被调用方（下游能力）
- 命令层：`commands/feature-dev.md` 定义 7 阶段流程与硬约束（例如先澄清再实现、实现前需用户批准）（`plugins/feature-dev/commands/feature-dev.md:57-69,85-93`）。
- 代理层：Phase 2/4/6 分别拉起 `code-explorer`、`code-architect`、`code-reviewer` 并行分析（`plugins/feature-dev/commands/feature-dev.md:41-54,78-82,106-109`；`plugins/feature-dev/README.md:61-65,118-125,181-185`）。

简化链路：
`marketplace source` -> `plugin root` -> `.claude-plugin/plugin.json` -> `commands/agents 自动发现` -> `/feature-dev` 编排执行 -> 并行子代理输出

### B. 数据结构

本目录数据结构为单文件 JSON manifest：

```json
{
  "name": "feature-dev",
  "version": "1.0.0",
  "description": "Comprehensive feature development workflow with specialized agents for codebase exploration, architecture design, and quality review",
  "author": {
    "name": "Sid Bidasaria",
    "email": "sbidasaria@anthropic.com"
  }
}
```

字段语义：
- `name`：插件唯一标识，要求 kebab-case（`manifest-reference.md:15-36`）。
- `version`：语义化版本（`manifest-reference.md:42-64`）。
- `description`：插件用途摘要（`manifest-reference.md:65-84`）。
- `author`：归属和联系信息（`manifest-reference.md:87-114`）。

### C. 协议与命令

- 协议层：这是“插件装配协议”的入口文件，系统先认 manifest，再扫描默认组件目录（`plugin-structure/SKILL.md:41-48,343-349`）。
- 命令层：本目录不包含命令实现，但其下游命令是 `/feature-dev`（`plugins/feature-dev/README.md:19-33`）。
- 代理层：本目录不包含 agent 实现，但其下游 3 个 agent 由命令按阶段调度（`plugins/feature-dev/README.md:251-307`）。

## 关键代码路径与文件引用

### 目标目录
- `plugins/feature-dev/.claude-plugin/plugin.json:1-9`

### 调用方（上游）
- `.claude-plugin/marketplace.json:62-70`
- `plugins/README.md:20,49-61,67-70`
- `plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:7-10,15-31`
- `plugins/plugin-dev/skills/plugin-structure/references/component-patterns.md:11-16,23-27`
- `plugins/plugin-dev/agents/plugin-validator.md:51-66,176-179`

### 被调用方（下游）
- `plugins/feature-dev/commands/feature-dev.md:1-125`
- `plugins/feature-dev/agents/code-explorer.md:1-51`
- `plugins/feature-dev/agents/code-architect.md:1-34`
- `plugins/feature-dev/agents/code-reviewer.md:1-46`
- `plugins/feature-dev/README.md:17-33,35-220,249-314,363-367`

### 配置、测试、脚本、文档上下文
- 配置：本目录 manifest + marketplace 条目（`.claude-plugin/marketplace.json`）+ 命令/agent frontmatter（`plugins/feature-dev/commands/*.md`, `plugins/feature-dev/agents/*.md`）。
- 测试：`plugins/feature-dev` 下未提供专用自动化测试目录或测试脚本（`find plugins/feature-dev -maxdepth 3 -type f` 仅 6 个文档/配置文件）。
- 脚本：目录内无执行脚本，属于提示协议型插件。
- 文档：插件说明主要在 `plugins/feature-dev/README.md`；仓库入口在 `README.md:48-50` 与 `plugins/README.md:11-27`。

## 依赖与外部交互

### 仓库内依赖
- 依赖 marketplace `source` 正确指向插件根（`.claude-plugin/marketplace.json:69`）。
- 依赖 Claude Code 自动发现机制先读取 manifest，再加载 `commands/`、`agents/`（`plugin-structure/SKILL.md:343-347`）。
- 依赖命令与 agent 文件命名/结构符合约定，才能被扫描和执行（`plugins/README.md:53-60`）。

### 外部交互（间接）
本目录本身不直接进行网络或系统调用；外部交互发生在下游 agent 执行阶段：
- 三个 agent 都声明 `WebSearch`、`WebFetch`，可与外部网页交互（`plugins/feature-dev/agents/code-explorer.md:4`，`code-architect.md:4`，`code-reviewer.md:4`）。
- 三个 agent 都声明 `BashOutput`，`code-reviewer` 默认审查 `git diff`，对 Git 工作区有运行时依赖（`plugins/feature-dev/agents/code-reviewer.md:13`）。

### 一致性依赖
`plugin.json` 与 marketplace 都维护作者与描述信息，存在双源一致性要求。当前作者名存在轻微差异：
- manifest：`Sid Bidasaria`（`plugins/feature-dev/.claude-plugin/plugin.json:6`）
- marketplace：`Siddharth Bidasaria`（`.claude-plugin/marketplace.json:66`）

## 风险、边界与改进建议

### 风险
1. 单点失效风险：该目录只有 `plugin.json`，缺失或 JSON 非法会导致整个插件无法进入发现链路。
2. 元数据漂移风险：marketplace 与 manifest 双写 `version/description/author`，长期维护可能不一致。
3. 自动化校验缺口：仓库内未见针对 `plugins/*/.claude-plugin/plugin.json` 的统一 CI 校验脚本，manifest 质量主要依赖人工审查。

### 边界
1. 本目录不实现业务逻辑，不包含命令、agent、hook、脚本。
2. 本目录只负责“可发现性”和“元信息声明”，不负责 7 阶段流程本身的正确性。
3. 运行行为、工具调用、外部交互全部由下游 `commands/` 与 `agents/` 决定。

### 改进建议
1. 增加 manifest CI 校验：统一校验 JSON 语法、`name` 格式、`version` 语义化、`author` 结构。
2. 增加 marketplace 一致性检查：对比 marketplace 条目与目标插件 `plugin.json` 的关键字段，阻断双源漂移。
3. 增加“发现链路 smoke test”：对每个插件做一次最小加载验证（manifest 可读、默认目录可扫描、关键命令可见）。
4. 统一作者展示名：将 `Sid` 与 `Siddharth` 对齐，减少归属审计歧义。
