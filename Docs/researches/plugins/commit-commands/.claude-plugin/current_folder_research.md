# plugins/commit-commands/.claude-plugin 目录研究（DIR）

## 场景与职责

`plugins/commit-commands/.claude-plugin` 是 `commit-commands` 插件的 manifest 目录，当前仅包含一个文件：`plugin.json`（`plugins/commit-commands/.claude-plugin/plugin.json:1-9`）。

该目录不实现 git 操作本身，而是作为插件系统的“发现与身份入口”：
- Claude Code 插件结构要求 manifest 位于 `.claude-plugin/plugin.json`，否则插件不会被识别（`plugins/README.md:49-55`，`plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:7-10`）。
- 组件发现阶段会先读取每个已启用插件的 manifest，再发现 commands/agents/skills/hooks（`plugins/plugin-dev/skills/plugin-structure/references/component-patterns.md:11-16`，`plugins/plugin-dev/skills/plugin-structure/SKILL.md:343-348`）。
- 本仓库 marketplace 通过 `source: "./plugins/commit-commands"` 指向插件根目录，随后才会读取该目录 manifest（`.claude-plugin/marketplace.json:39-49`）。

因此，这个目录的核心职责是：
1. 声明插件身份与元数据（name/version/author/description）。
2. 作为插件后续命令能力（`/commit`、`/commit-push-pr`、`/clean_gone`）被装配的前置条件。
3. 为 marketplace 展示信息和本地插件实体建立一致性锚点。

## 功能点目的

### 1) 声明插件唯一身份
`plugin.json` 当前字段：`name`、`description`、`version`、`author`（`plugins/commit-commands/.claude-plugin/plugin.json:2-8`）。
其中 `name` 是插件唯一标识，承担识别与冲突检测语义（`plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:15-31`）。

### 2) 打通 marketplace 到插件实体的路由
marketplace 条目定义了插件目录与展示元信息（`.claude-plugin/marketplace.json:40-48`），而本目录 `plugin.json` 是运行时真正读取的实体声明（`plugins/commit-commands/.claude-plugin/plugin.json:1-9`）。

### 3) 启用默认组件自动发现
当前 manifest 未声明自定义 `commands/agents/skills` 路径，意味着依赖默认自动发现目录（`plugins/plugin-dev/skills/plugin-structure/SKILL.md:344-348,355`）。
对本插件来说，即默认发现：
- `commands/commit.md`
- `commands/commit-push-pr.md`
- `commands/clean_gone.md`

### 4) 承载发布与归属信息
manifest 与 marketplace 都维护 `version/author`，用于展示和归因（`plugins/commit-commands/.claude-plugin/plugin.json:4-8`，`.claude-plugin/marketplace.json:42-46`）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### A. 关键流程（调用链）
1. 上游调用方：插件市场注册与发现器
- marketplace 注册插件 `commit-commands` 并指向源码目录（`.claude-plugin/marketplace.json:39-49`）。
- 启用阶段，系统读取 `.claude-plugin/plugin.json` 进入组件发现流程（`plugins/plugin-dev/skills/plugin-structure/references/component-patterns.md:9-16`）。

2. 目标对象：本目录 manifest
- `plugins/commit-commands/.claude-plugin/plugin.json` 提供插件最小元数据（`plugins/commit-commands/.claude-plugin/plugin.json:1-9`）。

3. 下游被调用方：命令协议文件与外部 CLI
- `commands/*.md` 被自动发现并作为 slash command 装配（`plugins/plugin-dev/skills/plugin-structure/SKILL.md:344`）。
- `/commit` 调用 git add/status/commit（`plugins/commit-commands/commands/commit.md:2`）。
- `/commit-push-pr` 调用 git + `gh pr create`（`plugins/commit-commands/commands/commit-push-pr.md:2,19`）。
- `/clean_gone` 执行分支/工作树清理命令（`plugins/commit-commands/commands/clean_gone.md:14,22,29-40`）。

简化链路：
`marketplace(source)` -> `plugin root` -> `.claude-plugin/plugin.json` -> `commands 自动发现` -> `/commit | /commit-push-pr | /clean_gone` 执行。

### B. 数据结构
本目录唯一关键数据结构是 `plugin.json`：

```json
{
  "name": "commit-commands",
  "description": "Streamline your git workflow with simple commands for committing, pushing, and creating pull requests",
  "version": "1.0.0",
  "author": {
    "name": "Anthropic",
    "email": "support@anthropic.com"
  }
}
```

字段语义：
- `name`：插件 ID（required，kebab-case，`manifest-reference.md:15-36`）。
- `version`：语义化版本（`manifest-reference.md:42-47`）。
- `description`：插件用途说明（`manifest-reference.md:65-77`）。
- `author`：归属与联系方式（`manifest-reference.md:87-114`）。

### C. 协议与命令（与本目录的关系）
本目录不直接定义命令协议，但它决定命令协议是否会被加载：
- 命令 frontmatter 协议：`description` 与 `allowed-tools`（`plugins/commit-commands/commands/commit.md:1-4`，`plugins/commit-commands/commands/commit-push-pr.md:1-4`）。
- 动态上下文注入协议：`!` 反引号命令读取运行时 git 状态（`plugins/commit-commands/commands/commit.md:8-11`，`plugins/commit-commands/commands/commit-push-pr.md:8-10`）。
- 执行约束协议：要求在单条消息中完成全部工具调用（`plugins/commit-commands/commands/commit.md:17`，`plugins/commit-commands/commands/commit-push-pr.md:20`）。

结论：`.claude-plugin` 是“装配入口协议”，`commands/*.md` 是“执行协议”。

## 关键代码路径与文件引用

### 目标目录直接对象
- `plugins/commit-commands/.claude-plugin/plugin.json:1-9`

### 调用方（上游）
- `.claude-plugin/marketplace.json:39-49`（插件市场注册）
- `plugins/README.md:18,47-61`（插件索引和标准结构）
- `plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:7-10`（manifest 必须路径）
- `plugins/plugin-dev/skills/plugin-structure/references/component-patterns.md:9-16`（发现阶段顺序）

### 被调用方（下游）
- `plugins/commit-commands/commands/commit.md:1-17`
- `plugins/commit-commands/commands/commit-push-pr.md:1-20`
- `plugins/commit-commands/commands/clean_gone.md:1-53`
- `plugins/commit-commands/README.md:1-225`

### 配置、测试、脚本、文档上下文
- 配置：manifest + marketplace + command frontmatter（`plugins/commit-commands/.claude-plugin/plugin.json:1-9`，`.claude-plugin/marketplace.json:39-49`，`plugins/commit-commands/commands/*.md`）。
- 测试：`plugins/commit-commands` 目录仅包含 5 个文件（manifest、README、3 个命令），未发现 `test/spec` 自动化测试文件。
- 脚本：无独立 `scripts/`；`clean_gone` 的 shell 管道逻辑内联在命令文档（`plugins/commit-commands/commands/clean_gone.md:29-40`）。
- 文档：插件 README 负责用户用法、依赖与排障说明（`plugins/commit-commands/README.md:86-89,179-210`）。

## 依赖与外部交互

### 仓库内依赖
1. 依赖 marketplace 的 `source` 指向插件根目录（`.claude-plugin/marketplace.json:47`）。
2. 依赖插件发现机制读取 `.claude-plugin/plugin.json`（`plugins/plugin-dev/skills/plugin-structure/SKILL.md:343-348`）。
3. 依赖默认命令目录扫描将 `commands/*.md` 注册为可调用命令（`plugins/plugin-dev/skills/plugin-structure/SKILL.md:344,355`）。

### 外部交互（间接）
本目录本身不直接调用外部系统；外部交互由下游命令触发：
- Git CLI：状态读取、提交、分支/工作树变更。
- GitHub CLI：`gh pr create`（`plugins/commit-commands/commands/commit-push-pr.md:2,19`）。
- 远端仓库与认证依赖：`origin` 远程、`gh auth login`（`plugins/commit-commands/README.md:86-88,195-202`）。

### 一致性依赖
manifest 与 marketplace 维护了重复元数据（name/description/version/author），需保持同步，避免展示与运行实体信息漂移（`plugins/commit-commands/.claude-plugin/plugin.json:2-8`，`.claude-plugin/marketplace.json:40-46`）。

## 风险、边界与改进建议

### 风险
1. 单点失效
- 本目录只有一个 `plugin.json`；一旦路径错误或 JSON 非法，插件会在发现阶段整体失效。

2. 元数据双源漂移
- `plugin.json` 与 `marketplace.json` 双份维护作者/版本/描述，存在长期不一致风险。

3. 自动化校验不足
- 当前仓库未见针对 `plugins/*/.claude-plugin/plugin.json` 的显式 CI 校验流程；问题更多在运行时暴露。

4. 下游能力边界与文档承诺可能漂移
- 该目录会放行命令装配，但不约束下游命令是否与 README 承诺完全一致；例如 `/commit` 的“避免提交 secrets”是文档承诺，命令文本未见硬性检查步骤（`plugins/commit-commands/README.md:44`，`plugins/commit-commands/commands/commit.md:15-17`）。

### 边界
1. 本目录仅负责 manifest 元数据，不定义命令工具白名单、执行步骤或安全策略。
2. 本目录不直接执行 git/gh，不直接联网，不包含脚本执行逻辑。
3. 行为正确性完全依赖下游命令 markdown 与宿主 Claude Code 执行器。

### 改进建议
1. 增加 manifest 级 CI 校验
- 对 `plugins/*/.claude-plugin/plugin.json` 做 JSON 语法 + 必填字段 + 命名规则校验（可参考 validator 流程，`plugins/plugin-dev/agents/plugin-validator.md:51-66`）。

2. 增加 marketplace-manifest 一致性检查
- 在 CI 自动比对 `source` 指向插件的 `name/version/author/description`，避免双源漂移。

3. 为 commit-commands 增加轻量 smoke test
- 至少校验三条命令文件存在、frontmatter 可解析、关键 `allowed-tools` 未丢失。

4. 明确发布同步清单
- 在插件发布流程中增加“先改 manifest 再改 marketplace（或反向）”的 checklist，降低人工漏改。
