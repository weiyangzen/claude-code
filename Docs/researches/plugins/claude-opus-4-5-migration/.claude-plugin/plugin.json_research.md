# plugins/claude-opus-4-5-migration/.claude-plugin/plugin.json 研究

## 场景与职责

`plugins/claude-opus-4-5-migration/.claude-plugin/plugin.json` 是该插件的 manifest 入口，职责是让 Claude Code 把该目录识别为可加载插件，并提供插件基础元数据，而不是直接执行迁移逻辑（`plugins/claude-opus-4-5-migration/.claude-plugin/plugin.json:1-9`）。

结合上下文，它处在一条明确的装配链路中：

1. 仓库 marketplace 用 `source: ./plugins/claude-opus-4-5-migration` 将插件暴露给安装与启用流程（`.claude-plugin/marketplace.json:18-27`）。
2. 插件加载阶段先读取 `.claude-plugin/plugin.json`，再做组件发现（`plugins/plugin-dev/skills/plugin-structure/SKILL.md:339-348`，`plugins/plugin-dev/skills/plugin-structure/references/component-patterns.md:9-16`）。
3. 本插件是“纯 Skill 型”结构：插件目录仅 5 个文件，不含 commands/agents/hooks/scripts（`find plugins/claude-opus-4-5-migration -maxdepth 5 -type f`），因此 manifest 成功加载后，下游主要是 `skills/claude-opus-4-5-migration/SKILL.md` 被发现与触发（`plugins/claude-opus-4-5-migration/skills/claude-opus-4-5-migration/SKILL.md:1-4`）。

## 功能点目的

1. 插件身份与唯一标识
- `name: "claude-opus-4-5-migration"` 提供唯一身份（`plugins/claude-opus-4-5-migration/.claude-plugin/plugin.json:2`）。
- 按 manifest 规范，`name` 参与插件识别与冲突检测，应满足 kebab-case（`plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:15-40`）。

2. 版本管理与发布语义
- `version: "1.0.0"` 提供语义版本基础（`plugins/claude-opus-4-5-migration/.claude-plugin/plugin.json:3`）。
- marketplace 中同名条目也声明 `version: "1.0.0"`，说明当前采用“双处同步”发布信息（`.claude-plugin/marketplace.json:18-21`）。

3. 能力描述与安装可见性
- `description` 与 README 首句一致，描述迁移范围为 Sonnet 4.x / Opus 4.1 -> Opus 4.5（`plugins/claude-opus-4-5-migration/.claude-plugin/plugin.json:4`，`plugins/claude-opus-4-5-migration/README.md:3`）。
- 插件总览表中同样把此插件声明为 skill 型能力（`plugins/README.md:16`）。

4. 作者归属
- `author.name/email` 作为归属与联系元信息（`plugins/claude-opus-4-5-migration/.claude-plugin/plugin.json:5-8`），并与 marketplace 保持一致（`.claude-plugin/marketplace.json:21-24`）。

5. 通过“未声明自定义路径”维持最小配置
- manifest 未设置 `commands/agents/hooks/mcpServers`，等同于依赖默认自动发现路径与默认行为（`plugins/plugin-dev/skills/plugin-structure/SKILL.md:86-100,343-355`，`plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:352-371`）。
- 对本插件而言，默认发现到的有效组件主要是 `skills/`（`plugins/claude-opus-4-5-migration/skills/claude-opus-4-5-migration/SKILL.md:1-4`）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 数据结构

目标文件是标准 JSON 对象（`plugins/claude-opus-4-5-migration/.claude-plugin/plugin.json:1-9`），字段包括：
- `name`（必需）
- `version`
- `description`
- `author`（对象：`name/email`）

它符合 manifest 参考文档定义的核心元数据字段集合（`plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:13-114`）。

### 2) 关键流程（调用方 -> 目标 -> 被调用方）

流程 A：安装/启用时的入口识别
1. 上游注册：marketplace 条目提供 `source` 到插件根目录（`.claude-plugin/marketplace.json:18-27`）。
2. 运行时读取：Claude Code 读取 `.claude-plugin/plugin.json`（`plugins/plugin-dev/skills/plugin-structure/references/component-patterns.md:11-16`）。
3. 自动发现：扫描默认 `skills/` 等目录（`plugins/plugin-dev/skills/plugin-structure/SKILL.md:343-348`）。

流程 B：能力触发到迁移执行
1. 用户按 README 示例给出迁移意图，如 “Migrate my codebase to Opus 4.5”（`plugins/claude-opus-4-5-migration/README.md:11-13`）。
2. Claude 匹配 skill frontmatter（`name/description`）并装载迁移指南（`plugins/claude-opus-4-5-migration/skills/claude-opus-4-5-migration/SKILL.md:1-4`）。
3. skill 执行迁移步骤：搜索模型串、替换为 Opus 4.5、移除不支持 beta header、按需做 prompt 调整并总结变更（`plugins/claude-opus-4-5-migration/skills/claude-opus-4-5-migration/SKILL.md:10-105`）。

### 3) 协议与命令面

- Manifest 协议：JSON（目标文件）。
- Skill 协议：Markdown + YAML frontmatter（`SKILL.md`）。
- 参考资料协议：Markdown（`references/effort.md`、`references/prompt-snippets.md`）。
- 命令面：本插件不提供 slash command；触发入口是自然语言意图（`plugins/claude-opus-4-5-migration/README.md:9-13`，`plugins/README.md:16`）。

### 4) 配置、脚本、测试、文档的实现关系

- 配置：`plugin.json`（插件元信息）+ `SKILL.md`（行为策略）+ `references/*.md`（参数与修复片段）。
- 脚本：插件目录中没有 shell/python/node 脚本文件（`find` 结果仅 5 个 `.md/.json` 文件）。
- 测试：仓库内未见此插件的专用测试或 CI 用例；其正确性主要依赖运行时加载和文档约束。
- 文档：README 描述用途与触发语句；外链到 Anthropic prompt best practices（`plugins/claude-opus-4-5-migration/README.md:1-21`）。

## 关键代码路径与文件引用

核心对象：
- `plugins/claude-opus-4-5-migration/.claude-plugin/plugin.json:1-9`

上游调用方/注册入口：
- `.claude-plugin/marketplace.json:18-27`（市场索引到插件目录）
- `plugins/README.md:16,49-60,67-70`（插件能力声明与结构规范）

下游被调用方/能力实现：
- `plugins/claude-opus-4-5-migration/README.md:1-21`（用户侧入口文档）
- `plugins/claude-opus-4-5-migration/skills/claude-opus-4-5-migration/SKILL.md:1-105`（迁移主流程）
- `plugins/claude-opus-4-5-migration/skills/claude-opus-4-5-migration/references/effort.md:1-70`（effort 参数方案）
- `plugins/claude-opus-4-5-migration/skills/claude-opus-4-5-migration/references/prompt-snippets.md:1-106`（问题定向 prompt 片段）

插件协议与校验参考（仓库内“规范来源”）：
- `plugins/plugin-dev/skills/plugin-structure/SKILL.md:41-43,46-100,339-355`
- `plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:7-10,15-40,42-63,87-114,352-393`
- `plugins/plugin-dev/skills/plugin-structure/references/component-patterns.md:9-17`
- `plugins/plugin-dev/agents/plugin-validator.md:51-66,67-75,176-179`

兼容性演进证据：
- `CHANGELOG.md:233`（曾修复 `plugin.json` 含 marketplace-only 字段导致加载失败的问题）。

## 依赖与外部交互

1. 运行时依赖（仓库外）
- 该文件依赖 Claude Code 运行时进行 manifest 解析与组件注册；仓库内没有对应 loader 实现代码。

2. 仓库内配置依赖
- 依赖 marketplace `source` 指向正确路径，否则安装/发现链路断开（`.claude-plugin/marketplace.json:25`）。
- 依赖插件目录结构满足标准约定（`.claude-plugin/plugin.json` + `skills/`），否则自动发现失败（`plugins/README.md:49-60`）。

3. 对用户代码库的外部交互
- 真正的变更动作发生在 skill 执行期，会扫描并修改用户项目中的模型配置、beta headers 和 prompt 文本（`SKILL.md:10-105`），不在本 manifest 内执行。

4. 外部文档/平台依赖
- README 链接 Anthropic 官方 prompt 指南（`plugins/claude-opus-4-5-migration/README.md:17`）。
- `references/effort.md` 依赖 API beta flag `effort-2025-11-24` 的可用性（`references/effort.md:17`）。

## 风险、边界与改进建议

### 风险

1. 元数据双源维护风险
- `plugin.json` 与 marketplace 同时维护 `description/version/author`，存在漂移可能（`plugin.json:3-8` vs `.claude-plugin/marketplace.json:19-24`）。

2. 指南时效风险
- skill 中硬编码了具体模型版本串与 beta header（`SKILL.md:25,35-38,44-46`），若平台版本更新，迁移建议可能过期。

3. 规则表述不一致风险
- `SKILL.md` 第 4 步要求“Add effort parameter set to high”（`SKILL.md:15`），但文末又写 effort 配置“only if user requests it”（`SKILL.md:105`），执行标准存在歧义。

4. 可验证性不足
- 该插件无脚本/测试，manifest 改动后主要靠运行时试错；仓库中也没有对该插件的自动化 smoke test。

### 边界

1. `plugin.json` 仅负责插件识别与元数据，不承担迁移逻辑正确性。
2. 实际迁移质量取决于 skill 文本策略与模型执行，不由 manifest 直接保证。
3. 本插件不直接定义命令、hooks、MCP servers，因此不会在本地触发额外进程或命令执行。

### 改进建议

1. 加入 manifest 与 marketplace 一致性检查
- 在 CI 增加脚本校验 `name/version/description/author` 是否一致，减少双源漂移。

2. 增加插件级 smoke test
- 最小化验证：插件可被识别、`skills/claude-opus-4-5-migration/SKILL.md` 可被发现并触发。

3. 修正 effort 规则歧义
- 统一 `SKILL.md` 中“默认添加 effort”与“仅用户请求时配置 effort”两处表述。

4. 建立模型字符串版本更新机制
- 将模型串与 beta header 列表抽到集中版本表或定期校验任务，避免迁移建议过时。
