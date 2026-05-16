# plugins/agent-sdk-dev/.claude-plugin 目录研究（DIR）

## 场景与职责

`plugins/agent-sdk-dev/.claude-plugin` 是 `agent-sdk-dev` 插件的 manifest 容器目录，当前仅包含 `plugin.json`，职责是为 Claude Code 插件发现与注册阶段提供最小元数据入口。

从仓库上下文看，该目录位于插件结构的强约束路径中：
- 标准结构要求每个插件在 `.claude-plugin/plugin.json` 声明元数据，见 `plugins/README.md:47-60`、`plugins/README.md:67-70`。
- plugin 结构参考明确指出 manifest 必须位于插件根下 `.claude-plugin/`，否则不会被识别，见 `plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:7-10`。
- 组件生命周期文档说明 Claude Code 启动时先扫描已启用插件并读取 `.claude-plugin/plugin.json`，见 `plugins/plugin-dev/skills/plugin-structure/references/component-patterns.md:9-16`。

因此，该目录不是业务逻辑执行点，而是插件装配链路中的“身份与发现锚点”。

## 功能点目的

### 1. 插件身份声明（Identity）
`plugin.json` 提供插件唯一名 `name`，用于插件识别/冲突避免/命名空间语义；这与 manifest 参考中的必填规则一致（kebab-case + 唯一性约束），见 `plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:15-36`。

本目录实值：
- `name: "agent-sdk-dev"`
- `description: "Claude Agent SDK Development Plugin"`
- `version: "1.0.0"`
- `author.name/email`
见 `plugins/agent-sdk-dev/.claude-plugin/plugin.json:1-9`。

### 2. 与 marketplace 条目的对齐（Registration Consistency）
仓库 marketplace 通过 `source: "./plugins/agent-sdk-dev"` 指向插件根目录，使运行时可定位到本目录的 manifest，见 `.claude-plugin/marketplace.json:12-15`。

该关系说明：
- marketplace 是“插件源入口”；
- `.claude-plugin/plugin.json` 是“插件实例元数据入口”；
- 两者共同完成注册链路闭环。

### 3. 对插件组件自动发现的前置支撑
`agent-sdk-dev` 的命令/代理能力在 `commands/` 与 `agents/` 下实现（`/new-sdk-app`、`agent-sdk-verifier-ts/py`），但这些组件能否被纳入插件装配，前提是 manifest 可被发现，见 `plugins/README.md:15` 与 `plugins/plugin-dev/skills/plugin-structure/references/component-patterns.md:11-16`。

## 具体技术实现（关键流程/数据结构/协议/命令）

### A. 关键流程（调用方 -> 目标 -> 被调用方）

1. 调用方：插件市场与启用流程
- marketplace 注册 `agent-sdk-dev`：`.claude-plugin/marketplace.json:12-15`。
- 插件结构规范要求读取 `.claude-plugin/plugin.json` 作为入口：`plugins/plugin-dev/skills/plugin-structure/references/component-patterns.md:11-16`。

2. 目标对象：`plugins/agent-sdk-dev/.claude-plugin/plugin.json`
- 提供插件基础元数据：`plugins/agent-sdk-dev/.claude-plugin/plugin.json:1-9`。

3. 被调用方：插件根组件
- 命令：`plugins/agent-sdk-dev/commands/new-sdk-app.md:1-176`
- Agent：`plugins/agent-sdk-dev/agents/agent-sdk-verifier-ts.md:1-145`
- Agent：`plugins/agent-sdk-dev/agents/agent-sdk-verifier-py.md:1-140`

简化调用链：
`marketplace.json(source)` -> `plugin root` -> `.claude-plugin/plugin.json` -> 组件发现/注册 -> `commands/agents` 执行。

### B. 数据结构

本目录核心数据结构为 JSON manifest：

```json
{
  "name": "agent-sdk-dev",
  "description": "Claude Agent SDK Development Plugin",
  "version": "1.0.0",
  "author": {
    "name": "Ashwin Bhat",
    "email": "ashwin@anthropic.com"
  }
}
```

字段语义映射：
- `name`：插件唯一标识（必需）。
- `description`：插件用途摘要。
- `version`：语义化版本号。
- `author`：归属与联系信息。

对照通用参考：`name` 必需且遵循 kebab-case，`version/description/author` 为常用元信息字段，见 `plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:15-114`。

### C. 协议与命令层面的间接约束

`.claude-plugin` 目录不直接存放命令实现，但通过 manifest 参与协议装配，间接约束以下命令行为可被加载：
- `/new-sdk-app` 交互创建流程，包含逐问收集、在线查最新版本、安装与校验，见 `plugins/agent-sdk-dev/commands/new-sdk-app.md:27-176`。
- verifier agents 输出统一报告协议 `PASS | PASS WITH WARNINGS | FAIL`，见：
  - `plugins/agent-sdk-dev/agents/agent-sdk-verifier-ts.md:111-145`
  - `plugins/agent-sdk-dev/agents/agent-sdk-verifier-py.md:106-140`

换言之：manifest 是“装配协议入口”，命令/agent markdown 是“行为协议本体”。

## 关键代码路径与文件引用

### 目标目录与直接对象
- `plugins/agent-sdk-dev/.claude-plugin/plugin.json:1-9`

### 上游调用方（注册/发现）
- `.claude-plugin/marketplace.json:12-15`（本插件源注册）
- `plugins/README.md:47-60`（标准插件目录结构）
- `plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:7-10`（manifest 必须路径）
- `plugins/plugin-dev/skills/plugin-structure/references/component-patterns.md:11-16`（启动扫描与注册顺序）

### 下游被调用方（组件落点）
- `plugins/agent-sdk-dev/commands/new-sdk-app.md:1-176`
- `plugins/agent-sdk-dev/agents/agent-sdk-verifier-ts.md:1-145`
- `plugins/agent-sdk-dev/agents/agent-sdk-verifier-py.md:1-140`
- `plugins/agent-sdk-dev/README.md:11-23,50-115`（命令/agents 能力说明）

### 配置、脚本、测试、文档上下文
- 研究流程脚本：`.ops/generate_daily_research_todo.sh:1-42`
- 研究勾选清单：`Docs/researches/blueprint_checklist.md:27`
- 文档化插件指南：`plugins/README.md:1-77`

测试现状：仓库中未发现专门针对 `plugins/agent-sdk-dev/.claude-plugin` 的自动化测试或校验脚本（`find . -maxdepth 4 -type f \( -name '*test*' -o -name '*spec*' -o -name '*validate*' \)` 未返回该插件相关验证用例）。

## 依赖与外部交互

### 1. 仓库内依赖
- 依赖 marketplace 条目定位插件根：`.claude-plugin/marketplace.json:12-15`。
- 依赖 Claude Code 插件发现机制读取 manifest：`plugins/plugin-dev/skills/plugin-structure/references/component-patterns.md:11-16`。
- 依赖插件根目录组件（`commands/agents`）提供实际功能。

### 2. 外部交互（间接）
本目录自身不发起外部请求，但其下游命令会触发外部交互：
- 文档抓取：`docs.claude.com`（overview/typescript/python）`plugins/agent-sdk-dev/commands/new-sdk-app.md:10-25`
- 版本源查询：`npmjs.com`、`pypi.org` `plugins/agent-sdk-dev/commands/new-sdk-app.md:74-79`
- 本地命令执行：`npm install`、`pip install`、`npx tsc --noEmit` `plugins/agent-sdk-dev/commands/new-sdk-app.md:83-87,119,164-166`

### 3. 配置一致性
- marketplace 条目未为 `agent-sdk-dev` 重复声明 `version/author`（仅 `name/description/source/category`），而这些信息在本目录 `plugin.json` 提供，见 `.claude-plugin/marketplace.json:12-16` 与 `plugins/agent-sdk-dev/.claude-plugin/plugin.json:1-9`。

## 风险、边界与改进建议

### 风险与边界
1. 单点 manifest 风险
- `.claude-plugin` 目录只有一个文件，若 `plugin.json` 缺失/损坏，整个插件会在发现阶段失效。

2. 自动化校验缺失
- 当前未见针对本 manifest 的仓库级 schema 校验/CI 检查，错误主要在运行时暴露。

3. 元数据双源可能漂移
- marketplace 与 plugin.json 各持有部分元数据；若长期维护不一致，会影响展示与排障效率。

4. 行为与实现解耦导致可预测性边界
- 该目录只描述身份，不承载执行逻辑；实际行为由 markdown 指令驱动，稳定性依赖运行时代理遵循程度。

### 改进建议
1. 增加 manifest 静态校验脚本
- 在 CI 中对 `plugins/*/.claude-plugin/plugin.json` 进行 JSON schema 校验和关键字段 lint（`name`、`version`、`author`）。

2. 增加元数据一致性检查
- 新增脚本比对 marketplace 与 plugin.json 的 `name/description/version/author`，至少对齐关键字段。

3. 在 `agent-sdk-dev` README 增加 manifest 变更约束
- 明确“修改插件名/版本时需同步检查 marketplace 与文档引用”。

4. 为研究与运维流程补充可复用检查命令
- 例如提供 `rg`/`jq` 一键检查命令，减少人工巡检成本。
