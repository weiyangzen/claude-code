# plugins/commit-commands/.claude-plugin/plugin.json 研究

## 场景与职责

`plugins/commit-commands/.claude-plugin/plugin.json` 是 `commit-commands` 插件的 manifest（`plugins/commit-commands/.claude-plugin/plugin.json:1-9`）。

在 Claude Code 插件生命周期里，它处在“插件发现入口”位置：
- 插件目录规范要求 manifest 必须放在 `.claude-plugin/plugin.json`，否则插件不会被识别（`plugins/README.md:49-55`，`plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:7-10`）。
- 组件发现阶段先读取每个插件的 manifest，再扫描命令/代理/技能/Hook（`plugins/plugin-dev/skills/plugin-structure/SKILL.md:343-348`，`plugins/plugin-dev/skills/plugin-structure/references/component-patterns.md:11-15`）。
- 仓库 marketplace 通过 `source: "./plugins/commit-commands"` 把插件根目录接入插件集合（`.claude-plugin/marketplace.json:40-48`）。

因此该文件的职责不是执行 Git 命令，而是：
1. 提供插件身份与展示元数据。
2. 作为 `commands/*.md` 被自动发现和注册的前置条件。
3. 为 marketplace 条目、插件 README、运行时装配提供一致的元信息锚点。

## 功能点目的

`plugin.json` 当前结构如下（`plugins/commit-commands/.claude-plugin/plugin.json:1-9`）：

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

各字段目的：

1. `name`
- 作为插件唯一标识（manifest required core field），用于插件识别与冲突检测（`plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:15-31`）。
- 当前值 `commit-commands` 符合 kebab-case 规则（`manifest-reference.md:33-36`）。

2. `description`
- 面向用户/市场展示的功能摘要，说明插件提供 Git 提交、推送、建 PR 的自动化能力（`plugins/commit-commands/.claude-plugin/plugin.json:3`）。
- 与 marketplace 条目描述同属“展示层元信息”，但文本不完全一致，存在后续漂移空间（`.claude-plugin/marketplace.json:41`）。

3. `version`
- 插件发布版本锚点，采用语义化版本（`manifest-reference.md:42-53`）。
- 当前为 `1.0.0`（`plugins/commit-commands/.claude-plugin/plugin.json:4`），与 marketplace 条目一致（`.claude-plugin/marketplace.json:42`）。

4. `author`
- 归属与支持联系方式（`plugins/commit-commands/.claude-plugin/plugin.json:5-8`）。
- 便于 marketplace 展示和维护责任追踪（`manifest-reference.md:87-114`）。

额外观察：
- 该 manifest 未声明 `commands/agents/hooks/mcpServers` 自定义路径，意味着依赖默认目录自动发现（`plugins/plugin-dev/skills/plugin-structure/SKILL.md:344-348,355`；`plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:214-215,262,301`）。
- 对本插件即默认加载 `commands/commit.md`、`commands/commit-push-pr.md`、`commands/clean_gone.md`。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 关键流程

从“插件被纳入列表”到“命令真正执行”的关键链路如下：

1. marketplace 声明插件来源
- `.claude-plugin/marketplace.json` 中 `commit-commands` 条目将 `source` 指向 `./plugins/commit-commands`（`.claude-plugin/marketplace.json:40-48`）。

2. 插件发现读取 manifest
- 启用/加载阶段读取 `.claude-plugin/plugin.json`（`plugins/plugin-dev/skills/plugin-structure/SKILL.md:343`，`component-patterns.md:11`）。

3. 默认组件扫描
- 因 manifest 未覆写路径，系统按默认位置扫描 `commands/` 下 markdown 文件（`plugins/plugin-dev/skills/plugin-structure/SKILL.md:344-345`，`manifest-reference.md:356-361`）。

4. 命令注册与执行
- `/commit`、`/commit-push-pr`、`/clean_gone` 由对应文件驱动：
  - `plugins/commit-commands/commands/commit.md:1-17`
  - `plugins/commit-commands/commands/commit-push-pr.md:1-20`
  - `plugins/commit-commands/commands/clean_gone.md:1-53`

5. 外部工具调用发生在命令层
- `plugin.json` 本身不调用命令；外部交互由下游命令执行，例如 `git`、`gh pr create`（`commit-push-pr.md:2,19`）。

### 2) 数据结构

`plugin.json` 是扁平 metadata 对象 + `author` 子对象：
- 扁平字段：`name`、`description`、`version`
- 嵌套字段：`author.name`、`author.email`

当前未使用的可扩展字段（但规范支持）：
- `homepage`、`repository`、`license`、`keywords`（`manifest-reference.md:115-208`）
- `commands`、`agents`、`hooks`、`mcpServers`（`manifest-reference.md:209-309`）

这说明 `commit-commands` 选择了“最小 manifest + 默认目录约定”的实现策略。

### 3) 协议与命令关系

1. manifest 协议
- 固定路径约束：`.claude-plugin/plugin.json`（`manifest-reference.md:7-10`）。
- 路径解析与发现顺序：默认目录优先，再合并自定义路径（`manifest-reference.md:352-371`）。

2. command 协议（由下游文件承担）
- `commit.md` 与 `commit-push-pr.md` 使用 frontmatter `allowed-tools` 限制工具白名单（`commit.md:2`，`commit-push-pr.md:2`）。
- 通过 `!` 反引号注入运行时 Git 上下文（`commit.md:8-11`，`commit-push-pr.md:8-10`）。
- `clean_gone.md` 采用内联 shell 管道清理 `[gone]` 分支（`clean_gone.md:29-40`）。

3. 同名命令双源现象
- 仓库根还存在 `.claude/commands/commit-push-pr.md`，内容与插件版高度相似（`.claude/commands/commit-push-pr.md:1-19` vs `plugins/commit-commands/commands/commit-push-pr.md:1-20`）。
- 这不是 `plugin.json` 直接逻辑，但属于其上下文依赖中的“行为一致性风险点”。

## 关键代码路径与文件引用

### 目标对象
- `plugins/commit-commands/.claude-plugin/plugin.json:1-9`

### 调用方（上游）
- marketplace 注册：`.claude-plugin/marketplace.json:40-48`
- 插件结构标准：`plugins/README.md:47-60`
- 发现机制说明：`plugins/plugin-dev/skills/plugin-structure/SKILL.md:339-349`
- 生命周期说明：`plugins/plugin-dev/skills/plugin-structure/references/component-patterns.md:7-17`

### 被调用方（下游）
- `plugins/commit-commands/commands/commit.md:1-17`
- `plugins/commit-commands/commands/commit-push-pr.md:1-20`
- `plugins/commit-commands/commands/clean_gone.md:1-53`
- 用户文档：`plugins/commit-commands/README.md:1-225`

### 配置、测试、脚本、文档上下文

1. 配置
- 插件实体配置：`plugins/commit-commands/.claude-plugin/plugin.json`
- 市场聚合配置：`.claude-plugin/marketplace.json`
- 命令执行配置（frontmatter）：`plugins/commit-commands/commands/*.md`

2. 测试
- `plugins/commit-commands` 目录未提供独立自动化测试文件（无 `test/`、`spec/`、脚本化验证）。

3. 脚本
- 目录内无 `scripts/`；`/clean_gone` 的 shell 逻辑直接内联在 markdown 中（`clean_gone.md:25-40`）。

4. 文档
- 插件行为说明与依赖前提：`plugins/commit-commands/README.md`
- 插件系统规范参考：`plugins/README.md` 与 `plugins/plugin-dev/skills/plugin-structure/*`

## 依赖与外部交互

### 仓库内依赖

1. 对 marketplace 的目录路由依赖
- `source` 需要准确指向插件根，否则运行时不会读取目标 manifest（`.claude-plugin/marketplace.json:47`）。

2. 对自动发现机制依赖
- 因未配置自定义路径，完全依赖默认目录扫描规则（`SKILL.md:344-348`，`manifest-reference.md:356-361`）。

3. 对下游命令文件完整性依赖
- 即使 `plugin.json` 合法，若 `commands/*.md` 缺失或 frontmatter 异常，实际功能仍会退化。

### 外部交互（间接）

`plugin.json` 本身无外部调用；实际外部交互在命令层发生：
- Git CLI：`git add/status/commit/checkout/push/branch/worktree`（见命令文件）。
- GitHub CLI：`gh pr create`（`commit-push-pr.md:2,19`）。
- 远程依赖：`origin` 远端与 `gh auth login`（`README.md:86-89,195-202`）。

## 风险、边界与改进建议

### 风险

1. manifest 单点失效
- `plugin.json` 路径错误、JSON 语法错误或关键字段不合法，会在发现阶段直接导致插件不可用。

2. 元数据双源漂移
- `plugin.json` 与 `marketplace.json` 同时维护 `name/description/version/author`，容易出现“展示信息与插件实体不一致”。

3. 行为文档与命令实现漂移
- README 声称 `/commit` 会规避 secrets、遵循约定式提交等（`README.md:41-45`），但命令文件并未给出硬性检查步骤（`commit.md:15-17`）。

4. 命令定义分叉风险
- `.claude/commands/commit-push-pr.md` 与插件版并存，后续维护可能出现更新不同步。

5. 缺少显式自动化校验
- 目前该插件目录无独立测试/校验脚本，manifest 与命令问题更可能在运行时暴露。

### 边界

1. `plugin.json` 只负责插件元数据与可选路径声明，不承担具体 Git/GitHub 操作。
2. 安全边界（例如工具白名单）主要由命令 frontmatter 决定，而非 manifest。
3. 正确性依赖 Claude Code 插件加载器与命令执行器对约定的实现。

### 改进建议

1. 增加 manifest CI 校验
- 对 `plugins/*/.claude-plugin/plugin.json` 进行 JSON 语法、`name` 规则、`version` 语义化、字段类型校验。

2. 增加 marketplace-manifest 一致性检查
- 在 CI 比对 `name/version/author/description`，防止双源漂移。

3. 增加 commit-commands smoke 测试
- 最少校验三条命令文件存在、frontmatter 可解析、`allowed-tools` 策略符合预期。

4. 统一 `commit-push-pr` 单一来源
- 明确以插件版或 `.claude/commands` 版为主，另一处改为引用或删除，减少行为分叉。

5. 把 README 的承诺转为可验证约束
- 对 secrets 规避、commit message 规则等，补充到命令指令或校验步骤中，降低“文档说了但执行不保证”的差距。
