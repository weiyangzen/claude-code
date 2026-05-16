# plugins/agent-sdk-dev/.claude-plugin/plugin.json 研究

## 场景与职责

`plugins/agent-sdk-dev/.claude-plugin/plugin.json` 是 `agent-sdk-dev` 插件的 manifest（机器可读入口），职责不是实现业务流程，而是让 Claude Code 识别该目录是一个可加载插件，并为其提供基础元数据。

从仓库上下文看，它位于“插件发现链路”的核心位置：

1. 仓库 marketplace 条目把 `agent-sdk-dev` 指向 `./plugins/agent-sdk-dev`（`.claude-plugin/marketplace.json:12-15`）。
2. 插件发现机制会先读取每个插件的 `.claude-plugin/plugin.json`（`plugins/plugin-dev/skills/plugin-structure/references/component-patterns.md:11-16`，`plugins/plugin-dev/skills/plugin-structure/SKILL.md:343-349`）。
3. 只有 manifest 被识别后，`commands/` 与 `agents/` 才进入后续自动发现（`plugins/plugin-dev/skills/plugin-structure/SKILL.md:344-346`）。

当前目标文件定义了 4 类元信息字段（`plugins/agent-sdk-dev/.claude-plugin/plugin.json:1-9`）：
- `name`: `agent-sdk-dev`
- `description`: `Claude Agent SDK Development Plugin`
- `version`: `1.0.0`
- `author`: `name/email`

## 功能点目的

1. 插件身份标识
- `name` 是插件唯一标识，决定冲突检测和插件识别（`plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:15-31`）。
- 本文件中的 `name: "agent-sdk-dev"` 与 marketplace 条目名称保持一致（`plugins/agent-sdk-dev/.claude-plugin/plugin.json:2`，`.claude-plugin/marketplace.json:12`）。

2. 版本与归属元数据
- `version` 为插件发布与维护提供语义版本基础（`plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:42-63`）。
- `author` 提供归属、联系信息（`plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:87-114`）。
- 本插件 manifest 包含 `version/author`，但 marketplace 中该条目未重复声明这两项（仅 `name/description/source/category`），形成“元数据分布在两处”的维护形态（`.claude-plugin/marketplace.json:12-16`）。

3. 能力入口前置条件
- `plugin.json` 不直接定义 `/new-sdk-app` 或 verifier 逻辑，但它是这些能力被发现的前置入口。
- 目录能力在 README 中声明为 1 个命令 + 2 个 agents（`plugins/README.md:15`），具体实现在：
  - `plugins/agent-sdk-dev/commands/new-sdk-app.md:1-176`
  - `plugins/agent-sdk-dev/agents/agent-sdk-verifier-ts.md:1-145`
  - `plugins/agent-sdk-dev/agents/agent-sdk-verifier-py.md:1-140`

4. 与贡献规范对齐
- `plugins/README.md` 要求每个插件提供 `.claude-plugin/plugin.json`（`plugins/README.md:49-55,67-70`）。
- 本文件满足该硬性要求，并采用仓库中主流字段组合（`name/description/version/author`，可对照其他插件 manifest）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 数据结构

文件为纯 JSON 对象（`plugins/agent-sdk-dev/.claude-plugin/plugin.json:1-9`），当前未配置自定义组件路径（如 `commands`、`agents`、`hooks`、`mcpServers`）。

这意味着该插件依赖默认自动发现目录：
- `commands/` 自动发现命令（`plugins/plugin-dev/skills/plugin-structure/SKILL.md:112-115`）
- `agents/` 自动发现代理（`plugins/plugin-dev/skills/plugin-structure/SKILL.md:138-141`）

### 2) 关键流程（调用链）

流程 A：仓库注册到插件目录
1. marketplace 注册 `agent-sdk-dev` -> `./plugins/agent-sdk-dev`（`.claude-plugin/marketplace.json:12-15`）。
2. 运行时进入插件目录后读取 `.claude-plugin/plugin.json`（`plugins/plugin-dev/skills/plugin-structure/references/component-patterns.md:11-16`）。

流程 B：manifest 驱动能力发现
1. 读取 manifest 成功后，扫描默认 `commands/` 与 `agents/`（`plugins/plugin-dev/skills/plugin-structure/SKILL.md:343-346`）。
2. 注册 `/new-sdk-app` 命令与两个 verifier agent（对应文件见上）。
3. 用户触发 `/new-sdk-app` 后再进入该命令定义的实施流程（`plugins/agent-sdk-dev/commands/new-sdk-app.md:1-176`）。

流程 C：命令到 agent 的插件内编排
1. `/new-sdk-app` 在验证阶段按语言分支触发：
- TypeScript -> `agent-sdk-verifier-ts`
- Python -> `agent-sdk-verifier-py`
2. 触发点在命令文档中明确写出（`plugins/agent-sdk-dev/commands/new-sdk-app.md:128-135`）。

### 3) 协议与命令面

- Manifest 协议：JSON（目标文件）。
- 命令协议：Markdown + YAML frontmatter（`new-sdk-app.md:1-4`）。
- Agent 协议：Markdown + YAML frontmatter（`agent-sdk-verifier-ts.md:1-5`，`agent-sdk-verifier-py.md:1-5`）。

与目标对象关联最强的运行命令并不在 `plugin.json` 内声明，而在命令正文中约束执行，包括：
- TypeScript：`npm init -y`、`npm install @anthropic-ai/claude-agent-sdk@latest`、`npm list ...`、`npx tsc --noEmit`（`new-sdk-app.md:68,83-87,119`）
- Python：`pip install claude-agent-sdk`、`pip show claude-agent-sdk`（`new-sdk-app.md:84,87`）

### 4) 与测试/脚本的实现关系

- `plugins/agent-sdk-dev` 下没有插件自带测试或执行脚本（目录仅 5 个文件，且均为 `.md/.json`）。
- 因此该 manifest 的正确性主要依赖：
  1. 运行时加载结果（是否被识别并发现组件）。
  2. 规范文档约束（manifest 位置/字段格式）。
  3. 人工或通用验证流程（如 plugin-validator 的 manifest 检查规范，`plugins/plugin-dev/agents/plugin-validator.md:56-66`）。

## 关键代码路径与文件引用

核心对象：
- `plugins/agent-sdk-dev/.claude-plugin/plugin.json:1-9`

上游调用方/注册入口：
- `.claude-plugin/marketplace.json:12-15`（市场条目指向本插件目录）
- `plugins/README.md:15`（对外声明本插件能力）
- `plugins/README.md:49-55,67-70`（manifest 是标准结构与贡献要求）

被调用方/下游能力：
- `plugins/agent-sdk-dev/commands/new-sdk-app.md:1-176`
- `plugins/agent-sdk-dev/commands/new-sdk-app.md:128-135`（命令触发 verifier 分支）
- `plugins/agent-sdk-dev/agents/agent-sdk-verifier-ts.md:1-145`
- `plugins/agent-sdk-dev/agents/agent-sdk-verifier-py.md:1-140`

结构规范与发现机制（外部化到 plugin-dev 参考）：
- `plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:7-10,15-31,42-63,87-114`
- `plugins/plugin-dev/skills/plugin-structure/references/component-patterns.md:11-16`
- `plugins/plugin-dev/skills/plugin-structure/SKILL.md:41-43,112-115,138-141,343-349,355`

演进证据（plugin.json 加载兼容性）：
- `CHANGELOG.md:233`（提到 `plugin.json` 与 marketplace 字段兼容性修复）

研究工程链路（本次任务相关）：
- `Docs/researches/blueprint_checklist.md:155`
- `.ops/generate_daily_research_todo.sh:1-42`

## 依赖与外部交互

1. Claude Code 运行时依赖（仓库外实现）
- 目标文件是运行时插件识别入口；解析、注册、安装流程在 Claude Code 主程序，不在本仓库。

2. 仓库内配置依赖
- 依赖 marketplace 注册路径正确（`.claude-plugin/marketplace.json:12-15`）。
- 依赖默认组件目录结构正确（`commands/`、`agents/`）以完成自动发现。

3. 文档与生态依赖
- `/new-sdk-app` 强依赖外部文档与包仓库（docs.claude.com、npmjs、PyPI），但这些依赖由命令层承担（`new-sdk-app.md:10-25,74-79`），不是 manifest 直接承担。

4. 测试与脚本现状
- 目标插件目录未提供专门测试脚本或 CI 校验入口。
- 本仓库对研究任务的自动化仅体现在 `.ops` 脚本（例如每日 todo 生成），不验证插件 manifest 语义正确性。

## 风险、边界与改进建议

### 风险

1. 元数据双源维护风险
- marketplace 与 plugin manifest 分开维护，`agent-sdk-dev` 在 marketplace 未写 `version/author`，实际依赖 manifest 承担这部分信息（`.claude-plugin/marketplace.json:12-16` vs `plugins/agent-sdk-dev/.claude-plugin/plugin.json:1-9`）。

2. 纯声明式插件的可验证性风险
- 无本地执行脚本与自动化测试，manifest 变更是否影响可加载性通常要到运行时才暴露。

3. 规范漂移风险
- manifest 规范主要在文档化引用中（plugin-dev skill/reference），若运行时行为变化而文档未同步，可能出现“文档正确、运行偏差”。

4. 兼容性风险
- 变更历史显示 `plugin.json` 与 marketplace 字段组合曾出现加载问题（`CHANGELOG.md:233`），说明该层对字段兼容较敏感。

### 边界

1. `plugin.json` 只定义身份与元数据，不实现业务工作流。
2. `/new-sdk-app` 与 verifier 的执行质量由命令/agent 文本协议与运行时模型行为决定，不由 manifest 保证。
3. 本仓库缺少直接解析 manifest 的可执行代码路径，很多行为需通过外部 Claude Code 运行时推断。

### 改进建议

1. 增加 manifest 自动校验脚本并接入 CI
- 校验 JSON 语法、`name` 格式、`version` semver、`author` 结构、与 marketplace `name/source` 一致性。

2. 为 marketplace 建立“字段来源规则”
- 明确 `version/author` 统一来源（仅 marketplace 或仅 plugin manifest），减少展示与维护歧义。

3. 增加插件级 smoke test
- 最小测试目标：安装后能识别 `agent-sdk-dev`，并发现 `/new-sdk-app`、`agent-sdk-verifier-ts`、`agent-sdk-verifier-py` 三项能力。

4. 在 `agent-sdk-dev` README 增加 manifest 变更约束
- 增加“变更 `plugin.json` 后必须执行的检查列表”，降低误改导致的发现失败概率。
