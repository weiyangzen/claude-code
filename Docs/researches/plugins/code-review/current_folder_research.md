# plugins/code-review 目录研究（DIR）

## 场景与职责

`plugins/code-review` 是一个“PR 自动审查编排型插件”，核心入口是 `/code-review` 命令，目标是在 Claude Code 会话中对 GitHub Pull Request 执行高信号问题筛查，并按需落地到 PR 评论。

- 目录定位与能力声明：`plugins/README.md:17`
- 插件被 marketplace 发现的入口：`.claude-plugin/marketplace.json:29-37`
- 插件元数据：`plugins/code-review/.claude-plugin/plugin.json:1-9`

本目录仅包含 3 个文件（`README.md`、`commands/code-review.md`、`.claude-plugin/plugin.json`），没有可执行源码、脚本或测试；其“实现”本质上是命令提示词驱动的执行协议。

职责分层：
1. `commands/code-review.md`：定义审查流程、并发 agent 策略、评论输出协议、工具使用约束。
2. `README.md`：面向使用者说明目标、操作方式、故障排查、配置项。
3. `.claude-plugin/plugin.json`：插件身份与发布元信息（name/version/author）。

## 功能点目的

### 1) 审查前短路（减少无效运行）
- 对 closed/draft/trivial/already-reviewed PR 直接跳过，避免浪费审查计算与审阅者注意力。  
  证据：`plugins/code-review/commands/code-review.md:14-22`

### 2) 按变更路径收集规范上下文
- 收集与改动文件同路径或父路径相关的 `CLAUDE.md`，将“项目约束”纳入审查输入。  
  证据：`plugins/code-review/commands/code-review.md:24-27,32-34`

### 3) 多 agent 并行审查 + 二次验证
- 先并发发现问题，再对子集问题做“验证性复审”，目标是提高 precision、压低误报。  
  证据：`plugins/code-review/commands/code-review.md:30-57`

### 4) 双通道输出（终端 / PR）
- 默认输出到终端；`--comment` 时写入 PR 评论，支持本地预审与线上留痕两种模式。  
  证据：`plugins/code-review/README.md:29-33`，`plugins/code-review/commands/code-review.md:63-67`

### 5) Inline 评论协议化
- 通过 MCP inline comment 工具逐条落点评论，并强制去重、建议块使用边界。  
  证据：`plugins/code-review/commands/code-review.md:71-77`

## 具体技术实现（关键流程/数据结构/协议/命令）

### A. 关键流程（端到端调用链）

1. 插件发现与命令暴露
- marketplace 通过 `source: "./plugins/code-review"` 注册目录。  
  证据：`.claude-plugin/marketplace.json:29-37`
- 目录结构遵循 Claude Code 插件约定（`.claude-plugin/` + `commands/` + `README.md`）。  
  证据：`plugins/README.md:49-61`

2. `/code-review` 被触发后的执行编排
- Step 1：haiku agent 判断是否应跳过。`plugins/code-review/commands/code-review.md:14-22`
- Step 2：haiku agent 收集相关 `CLAUDE.md` 路径。`...:24-27`
- Step 3：sonnet agent 总结 PR 变更。`...:28`
- Step 4：4 个并行 agent 发现问题（2 个 CLAUDE.md 合规 + 2 个 bug/logic）。`...:30-40`
- Step 5：对步骤 4 的问题做并发验证，bugs/logic 用 Opus，规范违规用 Sonnet。`...:55`
- Step 6：仅保留验证通过的问题。`...:57`
- Step 7：输出终端摘要；若 `--comment` 则继续评论路径。`...:59-67`
- Step 8-9：内部整理评论列表后，通过 MCP 工具发 inline comment。`...:69-77`

3. `--comment` 分支行为
- 若无问题：用 `gh pr comment` 发“无问题”摘要模板。`plugins/code-review/commands/code-review.md:65,93-101`
- 若有问题：逐条发 inline comment，并要求 `confirmed: true`。`...:71`

### B. 协议与配置结构

1. 命令 frontmatter（权限与描述）
- `description: Code review a pull request`  
- `allowed-tools` 限制为特定 `gh` 子命令 + `mcp__github_inline_comment__create_inline_comment`。  
  证据：`plugins/code-review/commands/code-review.md:1-4`

2. 插件元数据（plugin manifest）
- 字段：`name`, `description`, `version`, `author`。  
  证据：`plugins/code-review/.claude-plugin/plugin.json:1-9`

3. 审查问题结构（隐式数据结构）
- 每个 issue 至少包含：问题描述 + 被标记原因（如 `CLAUDE.md adherence` / `bug`）。  
  证据：`plugins/code-review/commands/code-review.md:30`

4. 评论链接协议（严格格式）
- 必须使用 full SHA + `#Lx-Ly` 行区间，且要包含上下文行。  
  证据：`plugins/code-review/commands/code-review.md:103-109`

### C. 核心命令与工具交互

1. GitHub CLI（`gh`）
- 读取 PR 与评论：`gh pr view`（命令中显式提及 `--comments`）。`plugins/code-review/commands/code-review.md:18`
- 发汇总评论：`gh pr comment`。`...:65`
- diff / list / issue 相关能力在 `allowed-tools` 白名单中。`...:2`

2. MCP inline comment
- 用于逐问题精确行内评论（`confirmed: true`）。`plugins/code-review/commands/code-review.md:71`

3. 高信号过滤协议
- 明确列出“可报问题”与“禁止报问题”边界，避免泛化 code-quality 噪音。  
  证据：`plugins/code-review/commands/code-review.md:41-51,79-86`

## 关键代码路径与文件引用

### 目录内核心文件
- `plugins/code-review/commands/code-review.md:1-109`  
  主执行规范（流程、并发 agent、输出与评论协议、工具白名单）。
- `plugins/code-review/README.md:1-258`  
  用户使用说明、示例、阈值/策略文档、故障排查。
- `plugins/code-review/.claude-plugin/plugin.json:1-9`  
  元数据定义。

### 上下游与上下文依赖
- 插件市场注册：`.claude-plugin/marketplace.json:29-37`
- 插件总览与结构规范：`plugins/README.md:17,49-61`
- 仓库级插件入口说明：`README.md:48-50`

### 调用方与被调用方
- 调用方（上游）
1. Claude Code 插件发现机制（通过 marketplace + `commands/` 自动发现）。
2. 用户/自动流程触发 `/code-review`。

- 被调用方（下游）
1. GitHub CLI（`gh pr view/diff/list/comment` 等受白名单约束）。
2. MCP 工具 `mcp__github_inline_comment__create_inline_comment`。
3. 子代理模型层（haiku/sonnet/opus）用于不同阶段任务分工。

### 配置、测试、脚本、文档覆盖情况
- 配置：frontmatter `allowed-tools` + `--comment` 参数路径 + 文档化阈值策略。  
  证据：`plugins/code-review/commands/code-review.md:1-4,63-67`，`plugins/code-review/README.md:214-221`
- 测试：目录内无自动化测试文件（无 `test/`、无测试脚本）。
- 脚本：目录内无可执行脚本（无 `scripts/`）。
- 文档：有完整 README，且在 `plugins/README.md` 有目录级索引。

## 依赖与外部交互

### 运行时依赖
1. Claude Code 运行时的插件与命令执行框架。
2. GitHub CLI 安装与登录状态（`gh auth`）。
3. 可用的 GitHub 网络访问与仓库权限。
4. MCP `github_inline_comment` 能力可用。

### 外部系统交互
1. PR 元数据与 diff 读取：通过 `gh` 与 GitHub API 通道。
2. 评论写入：`gh pr comment`（摘要）与 MCP inline comment（逐问题）。
3. 仓库上下文：审查依据当前仓库 PR 状态及改动集合。

### 约束性协议
1. 不做工具探测调用（假定工具可用）。`plugins/code-review/commands/code-review.md:8-10`
2. 只在需要时调用工具，强调最小调用面。`...:10`
3. 不允许 web fetch，GitHub 交互限定 `gh`。`...:90`

## 风险、边界与改进建议

### 风险与边界

1. README 与命令规范存在实现语义漂移
- README 强调“0-100 评分 + 80 阈值过滤”。`plugins/code-review/README.md:23-25,78-83,216-219,239-243`
- 命令文件实际流程是“验证通过/未通过”过滤，未定义数值打分步骤。`plugins/code-review/commands/code-review.md:55-57`
- 影响：用户预期“阈值可调”但执行上可能并无显式分数输出。

2. “历史上下文分析”描述与执行指令不完全一致
- README 写明 agent #4 做 git blame/history 分析。`plugins/code-review/README.md:22,55,236`
- 命令 Step 4 的 agent #4 定义为“introduced code 问题扫描”，未要求 git blame 调用。`plugins/code-review/commands/code-review.md:38-40`
- 影响：文档承诺的分析深度可能高于实际执行。

3. 外部依赖集中在 GitHub 工具链
- `gh` 登录态、仓库权限、MCP 服务任一失效会影响审查闭环（特别是 `--comment` 场景）。

4. 无目录内自动化回归
- 该插件完全是提示词协议，缺少脚本化回归与一致性检查，prompt 演进后易出现行为漂移。

5. 对 `CLAUDE.md` 生态有依赖但非强制
- 命令明确收集 `CLAUDE.md`（若存在），无该文件时合规审查维度会弱化。`plugins/code-review/commands/code-review.md:24-26`

### 改进建议

1. 统一 README 与命令真实行为
- 二选一：  
  - 在命令中补全“显式打分 + 阈值过滤”步骤；或  
  - 在 README 中改为“验证制高信号过滤”描述。

2. 明确历史分析职责
- 若保留“history/blame”卖点，应在命令步骤中显式加入对应工具调用与判定规则。

3. 增加最小回归验证
- 新增轻量脚本验证关键不变量：  
  - frontmatter 语法正确  
  - 必要步骤（skip-check、并发审查、验证过滤、`--comment` 分支）存在  
  - 评论链接格式规则存在

4. 提供可观测性输出模板
- 在终端摘要增加结构化字段（issue count、validated count、skipped reason），便于 CI/日志消费。

5. 为权限声明做最小化审查
- 定期核对白名单命令与实际流程是否一致，避免权限冗余或能力缺失。
