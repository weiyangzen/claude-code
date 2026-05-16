# plugins/claude-opus-4-5-migration/.claude-plugin 目录研究（DIR）

## 场景与职责

`plugins/claude-opus-4-5-migration/.claude-plugin` 是该插件的 manifest 目录，当前仅包含 `plugin.json`（`plugins/claude-opus-4-5-migration/.claude-plugin/plugin.json:1`）。

在 Claude Code 插件体系里，这个目录承担的是“插件可识别入口”职责，而不是业务逻辑执行职责：
- 插件标准结构要求 manifest 必须放在 `.claude-plugin/plugin.json`，否则插件不会被识别（`plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:7-10`）。
- 组件发现生命周期第一步是读取每个已启用插件的 `plugin.json`（`plugins/plugin-dev/skills/plugin-structure/references/component-patterns.md:11-16`）。
- 本仓库 marketplace 通过 `source: "./plugins/claude-opus-4-5-migration"` 指向插件根，运行时再进入该目录读取 manifest（`.claude-plugin/marketplace.json:18-26`）。

结论：该目录是“注册与发现锚点”，控制插件是否能进入后续 skill 自动发现链路。

## 功能点目的

### 1. 插件身份与唯一标识
`plugin.json` 声明：
- `name: "claude-opus-4-5-migration"`
- `version: "1.0.0"`
- `description`
- `author`（name/email）
见 `plugins/claude-opus-4-5-migration/.claude-plugin/plugin.json:2-8`。

其中 `name` 是插件唯一标识，承担识别、冲突检测和命名空间语义（`plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:15-31`）。

### 2. marketplace 安装展示与路由一致性
仓库级 marketplace 条目重复声明了该插件的 name/description/version/author，并提供 source 路径（`.claude-plugin/marketplace.json:18-26`）。这使 `/plugin` 市场流程可以展示元数据并定位到正确目录，再读取本目录 manifest。

### 3. 下游能力装配前置条件
本插件的实际能力在 `skills/claude-opus-4-5-migration/SKILL.md` 与 `references/*` 中定义，不在 `.claude-plugin` 目录里实现（`find plugins/claude-opus-4-5-migration -maxdepth 4 -type f` 仅 5 个文件，其中本目录只有 1 个 manifest）。

因此 `.claude-plugin` 的目的不是“执行迁移”，而是“让迁移能力可被系统装配”。

## 具体技术实现（关键流程/数据结构/协议/命令）

### A. 关键流程（调用方 -> 目标 -> 被调用方）

1. 调用方（上游）
- marketplace 注册插件源：`.claude-plugin/marketplace.json:18-26`
- 插件发现机制读取 manifest：`plugins/plugin-dev/skills/plugin-structure/references/component-patterns.md:11-16`
- 插件结构与校验规则把 `.claude-plugin/plugin.json` 作为必检对象：
  - `plugins/plugin-dev/skills/plugin-structure/SKILL.md:41-48`
  - `plugins/plugin-dev/agents/plugin-validator.md:51-66`

2. 目标对象（本目录）
- `plugins/claude-opus-4-5-migration/.claude-plugin/plugin.json:1-9`

3. 被调用方（下游）
- 插件根 README 提供触发语句与外部文档链接：`plugins/claude-opus-4-5-migration/README.md:11-17`
- Skill 主流程（模型串替换、beta header 处理、按需 prompt 修复）：`plugins/claude-opus-4-5-migration/skills/claude-opus-4-5-migration/SKILL.md:10-105`
- 参考材料：
  - effort 参数：`plugins/claude-opus-4-5-migration/skills/claude-opus-4-5-migration/references/effort.md:15-56`
  - prompt 片段：`plugins/claude-opus-4-5-migration/skills/claude-opus-4-5-migration/references/prompt-snippets.md:1-106`

简化链路：
`marketplace source` -> `plugin root` -> `.claude-plugin/plugin.json` -> `skills 自动发现与触发` -> `对用户仓库执行迁移编辑`

### B. 数据结构

本目录核心结构是单文件 JSON manifest：

```json
{
  "name": "claude-opus-4-5-migration",
  "version": "1.0.0",
  "description": "Migrate your code and prompts from Sonnet 4.x and Opus 4.1 to Opus 4.5.",
  "author": {
    "name": "William Hu",
    "email": "whu@anthropic.com"
  }
}
```

字段语义：
- `name`：插件唯一标识（required，kebab-case）。
- `version`：语义化版本。
- `description`：插件用途摘要。
- `author`：归属与联系方式。

这些字段与 manifest 规范的核心/推荐字段一致（`plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:15-90`）。

### C. 协议与命令

- 本目录不承载 slash command / hook / agent 定义，只承载 manifest。
- 协议层面是“插件装配协议”：manifest 被读到后，Claude Code 才会继续扫描 `skills/` 并在匹配描述时加载 skill（`plugins/plugin-dev/skills/skill-development/SKILL.md:271-276`）。
- 命令层面：本插件没有自定义命令入口，主要通过自然语言触发 skill（`plugins/claude-opus-4-5-migration/README.md:11-13`）。

## 关键代码路径与文件引用

### 目标目录
- `plugins/claude-opus-4-5-migration/.claude-plugin/plugin.json:1-9`

### 调用方（上游）
- `.claude-plugin/marketplace.json:18-26`
- `plugins/README.md:16,47-54,67-70`
- `plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:7-10`
- `plugins/plugin-dev/skills/plugin-structure/references/component-patterns.md:11-16`
- `plugins/plugin-dev/agents/plugin-validator.md:51-66`

### 被调用方（下游）
- `plugins/claude-opus-4-5-migration/README.md:1-21`
- `plugins/claude-opus-4-5-migration/skills/claude-opus-4-5-migration/SKILL.md:1-105`
- `plugins/claude-opus-4-5-migration/skills/claude-opus-4-5-migration/references/effort.md:1-70`
- `plugins/claude-opus-4-5-migration/skills/claude-opus-4-5-migration/references/prompt-snippets.md:1-106`

### 配置、测试、脚本、文档上下文
- 配置：`plugin.json`（本目录） + marketplace 条目 + skill frontmatter。
- 测试：仓库内未见针对该目录的专用自动化测试文件。
- 脚本：该目录无执行脚本；校验流程主要依赖通用 validator 规范（`plugins/plugin-dev/agents/plugin-validator.md`）。
- 文档：插件 README 与 skill/references 构成完整说明链。

## 依赖与外部交互

### 仓库内依赖
- 依赖 marketplace 把 source 指向插件根（`.claude-plugin/marketplace.json:25`）。
- 依赖 Claude Code 插件发现机制读取 `.claude-plugin/plugin.json`（`component-patterns.md:11-16`）。
- 依赖 skill 自动发现机制加载 `skills/*/SKILL.md`（`skill-development/SKILL.md:271-276`）。

### 外部交互（间接）
本目录本身不发起外部调用；外部交互发生在其下游迁移目标中：
- 模型与平台目标字符串涉及 Anthropic API / AWS Bedrock / Google Vertex / Azure（`SKILL.md:33-38`）。
- README 引用外部提示词指南（`plugins/claude-opus-4-5-migration/README.md:17`）。

### 一致性依赖
`plugin.json` 与 marketplace 条目都维护了描述性元数据（description/version/author）。两处需保持一致，否则可能出现安装展示信息与本地 manifest 信息漂移。

## 风险、边界与改进建议

### 风险
1. 单点失效风险：该目录只有一个 `plugin.json`，一旦缺失或 JSON 非法，插件会在发现阶段失效。
2. 元数据双源漂移：marketplace 与 manifest 同时维护版本/作者/描述，长期可能不一致。
3. 自动化校验缺口：当前仓库未见对该目录 manifest 的 CI 强制 schema 校验。

### 边界
1. 本目录只负责元数据与发现入口，不执行实际迁移动作。
2. 不包含脚本、测试、hook、command；运行行为完全在下游 skill 指令里。
3. 对外部平台协议的适配（模型 ID、beta 头）不在本目录实现，只在下游文档策略中体现。

### 改进建议
1. 增加 manifest CI 校验：对 `plugins/*/.claude-plugin/plugin.json` 执行 JSON schema + 基础 lint（name/version/author）。
2. 增加 marketplace 一致性检查：自动对比 `source` 目标插件的 `plugin.json` 与 marketplace 元数据字段。
3. 在插件发布流程中引入“manifest smoke check”：至少验证 JSON 语法、name 合法性、source 可达性。
4. 在本插件 README 增补“manifest 变更同步项”说明，减少人工漏改。
