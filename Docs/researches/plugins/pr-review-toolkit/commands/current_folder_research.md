# plugins/pr-review-toolkit/commands 目录研究（DIR）

## 场景与职责

`plugins/pr-review-toolkit/commands` 是 `pr-review-toolkit` 插件的命令编排层，当前仅包含一个命令文件：
- `plugins/pr-review-toolkit/commands/review-pr.md`

该目录不是“静态分析器实现代码”，而是“流程协议定义”。它负责把 `/pr-review-toolkit:review-pr` 这个入口命令转成一套可执行的评审流程：
1. 接收用户参数（`$ARGUMENTS`）并识别评审范围（`review-pr.md:11,15-29`）。
2. 基于变更上下文（`git diff --name-only`、`gh pr view`）决定应触发哪些审查维度（`review-pr.md:30-44`）。
3. 编排 6 个专项 agent 串行或并行运行（`review-pr.md:45-56,109-113`）。
4. 聚合多 agent 输出并形成可执行行动清单（`review-pr.md:57-88`）。

在插件装配链路里的位置：
- marketplace 注册插件来源目录为 `./plugins/pr-review-toolkit`（`.claude-plugin/marketplace.json:117-125`）。
- 插件清单由 `.claude-plugin/plugin.json` 提供元信息（`plugins/pr-review-toolkit/.claude-plugin/plugin.json:1-9`）。
- Claude Code 自动扫描 `commands/*.md` 生成命令（`plugins/plugin-dev/skills/plugin-structure/SKILL.md:343-345`）。
- `review-pr.md` 文件名映射命令名，配合插件名形成 `/pr-review-toolkit:review-pr`（`plugins/plugin-dev/skills/plugin-structure/SKILL.md:307-309` + `plugins/README.md:25`）。

职责边界：
- 本目录负责“命令协议与调度策略”，不直接承载业务代码修改逻辑。
- 具体评审能力由 `agents/` 中的角色协议承担，本目录仅负责触发与汇总。

## 功能点目的

### 1. 统一 PR 评审入口并支持按方面筛选
目的：让用户既可一键全量评审，也可按问题类型定向评审。
- 支持 `comments/tests/errors/types/code/simplify/all`（`review-pr.md:22-29`）。
- 参数通过 `$ARGUMENTS` 注入（`review-pr.md:11`）。

### 2. 用“改动感知”减少无关审查
目的：把审查成本聚焦在真正变化的部分，避免“全仓泛扫”。
- 要求先读取 `git diff --name-only` 与 `gh pr view`（`review-pr.md:31-33`）。
- 再按改动类型映射适用 agent（`review-pr.md:37-43`）。

### 3. 将“质量审查”与“代码简化”拆阶段
目的：先保证正确性，再做可维护性抛光，避免过早优化。
- `code-simplifier` 明确定位在“通过审查后”阶段（`review-pr.md:43`）。

### 4. 提供串行/并行两种执行模式
目的：兼顾可读性（串行）与效率（并行）。
- 串行：便于逐个处理报告（`review-pr.md:47-51`）。
- 并行：缩短端到端时长（`review-pr.md:52-56,109-113`）。

### 5. 标准化结果出口，降低落地成本
目的：将多 agent 异构结果统一映射为可执行修复计划。
- 输出结构固定为 Critical / Important / Suggestions / Strengths / Recommended Action（`review-pr.md:67-88`）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### A. 关键流程（命令状态机）

1. 命令初始化
- frontmatter 定义 `description`、`argument-hint`、`allowed-tools`（`review-pr.md:1-5`）。
- `allowed-tools` 明确允许：`Bash`, `Glob`, `Grep`, `Read`, `Task`（`review-pr.md:4`）。

2. 评审范围解析
- 输入源：`$ARGUMENTS`（`review-pr.md:11`）。
- 默认语义：未指定时走 `all`（`review-pr.md:18,28`）。

3. 变更探测与上下文确认
- 执行 `git diff --name-only` 获取改动文件（`review-pr.md:31`）。
- 执行 `gh pr view` 判断 PR 上下文（`review-pr.md:32`）。

4. 评审面决策
- 规则映射：
  - Always: `code-reviewer`（`review-pr.md:38`）
  - 测试改动: `pr-test-analyzer`（`review-pr.md:39`）
  - 注释/文档改动: `comment-analyzer`（`review-pr.md:40`）
  - 错误处理改动: `silent-failure-hunter`（`review-pr.md:41`）
  - 类型改动: `type-design-analyzer`（`review-pr.md:42`）
  - 收尾优化: `code-simplifier`（`review-pr.md:43`）

5. 子 agent 调度
- 串行模式：按单个完整报告逐个推进（`review-pr.md:47-51`）。
- 并行模式：同时发起并统一回收（`review-pr.md:52-56`）。
- 并行触发示例通过 `all parallel` 体现（`review-pr.md:111-113`）。

6. 汇总与行动计划生成
- 合并为 4 类发现 + 1 个执行计划（`review-pr.md:59-88`）。
- 计划顺序是“先 critical，再 important，再建议，再复跑” （`review-pr.md:83-87`）。

### B. 隐含数据结构（协议级）

命令无显式 JSON schema，但从文本协议可推导关键结构：

1. `ReviewAspects`
- 枚举：`comments | tests | errors | types | code | simplify | all`
- 扩展标记：`parallel` 作为执行模式修饰词（`review-pr.md:109-113`）。

2. `ChangedContext`
- `changedFiles`: 来自 `git diff --name-only`（`review-pr.md:31`）。
- `prContext`: 来自 `gh pr view`（`review-pr.md:32`）。

3. `ActivatedAgents`
- 由“方面参数 + 改动映射”共同决定（`review-pr.md:35-44`）。

4. `AggregatedFindings`
- 统一分桶：`Critical Issues` / `Important Issues` / `Suggestions` / `Strengths`（`review-pr.md:71-82`）。

5. `ActionPlan`
- 固定 4 步执行顺序（`review-pr.md:83-87`）。

### C. 协议与命令约束

1. frontmatter 协议
- `argument-hint: "[review-aspects]"` 指导参数形态（`review-pr.md:3`）。
- `allowed-tools` 限定运行面，核心依赖 `Task` 进行子代理调度（`review-pr.md:4`）。

2. 子代理契约（被调用方）
- 命令侧引用 6 个 agent 名称（`review-pr.md:38-43,117-146`）。
- agent 侧定义了异构评分/分级机制：
  - `code-reviewer`: 0-100，且仅报告 `>=80`（`agents/code-reviewer.md:22-33`）
  - `pr-test-analyzer`: 1-10 关键度（`agents/pr-test-analyzer.md:27-48`）
  - `type-design-analyzer`: 四维 1-10 评分（`agents/type-design-analyzer.md:24-47,58-69`）
  - `silent-failure-hunter`: `CRITICAL/HIGH/MEDIUM`（`agents/silent-failure-hunter.md:103-109`）

3. 外部命令依赖
- `git diff --name-only`：本地仓库变更识别。
- `gh pr view`：PR 元数据获取（可能涉及网络与鉴权）。
- 证据：`review-pr.md:31-33`。

### D. 调用方与被调用方关系

1. 调用方（上游）
- 用户 slash command 入口：`/pr-review-toolkit:review-pr`（`review-pr.md:94-113`）。
- 插件目录索引文档暴露该命令（`plugins/README.md:25`）。
- marketplace + plugin manifest 提供发现与装配（`.claude-plugin/marketplace.json:117-125`，`plugins/pr-review-toolkit/.claude-plugin/plugin.json:1-9`）。

2. 被调用方（下游）
- `comment-analyzer`（`agents/comment-analyzer.md:1-70`）
- `pr-test-analyzer`（`agents/pr-test-analyzer.md:1-69`）
- `silent-failure-hunter`（`agents/silent-failure-hunter.md:1-130`）
- `type-design-analyzer`（`agents/type-design-analyzer.md:1-110`）
- `code-reviewer`（`agents/code-reviewer.md:1-47`）
- `code-simplifier`（`agents/code-simplifier.md:1-83`）

### E. 配置/测试/脚本/文档上下文

1. 配置
- 插件级配置：`plugins/pr-review-toolkit/.claude-plugin/plugin.json`。
- 命令级配置：`review-pr.md` frontmatter。
- agent 级配置：`agents/*.md` frontmatter（`name/model/color/description`）。

2. 测试
- `plugins/pr-review-toolkit/commands` 无测试文件。
- 全插件范围也无 `*.spec.*` / `*.test.*` 可执行测试资产；仅存在命名包含 `test` 的 agent 说明文件 `pr-test-analyzer.md`。

3. 脚本
- 目标插件目录内不存在 `.sh/.py/.js/.ts` 脚本，命令行为完全依赖 Markdown 协议与运行时工具。

4. 文档
- 插件 README 给出完整能力说明、触发语句、流程建议与排障（`plugins/pr-review-toolkit/README.md:1-313`）。
- 该 README 与命令示例在能力枚举上保持一致（`README.md:157-170` 对应 `review-pr.md:20-29,94-113`）。

## 关键代码路径与文件引用

### 目标目录
- `plugins/pr-review-toolkit/commands/review-pr.md:1-189`

### 上游入口与装配
- `.claude-plugin/marketplace.json:117-125`
- `plugins/pr-review-toolkit/.claude-plugin/plugin.json:1-9`
- `plugins/README.md:25`
- `plugins/plugin-dev/skills/plugin-structure/SKILL.md:307-309,343-345`

### 下游被调用方
- `plugins/pr-review-toolkit/agents/comment-analyzer.md:1-70`
- `plugins/pr-review-toolkit/agents/pr-test-analyzer.md:1-69`
- `plugins/pr-review-toolkit/agents/silent-failure-hunter.md:1-130`
- `plugins/pr-review-toolkit/agents/type-design-analyzer.md:1-110`
- `plugins/pr-review-toolkit/agents/code-reviewer.md:1-47`
- `plugins/pr-review-toolkit/agents/code-simplifier.md:1-83`

### 相关文档
- `plugins/pr-review-toolkit/README.md:1-313`
- `plugins/README.md:25`

### 测试与脚本现状
- `plugins/pr-review-toolkit/commands/` 当前仅 `review-pr.md`。
- 插件范围未发现可执行测试与脚本文件（除 Markdown/JSON 外）。

## 依赖与外部交互

### 内部依赖
1. 插件自动发现机制：依赖标准目录结构与 manifest（`plugin-structure/SKILL.md:341-349`）。
2. agent 命名一致性：命令引用名必须与 `agents/*.md` frontmatter 的 `name` 一致。
3. 项目规范上下文：多个 agent 依赖 `CLAUDE.md` 指南做判断（如 `code-reviewer.md:8,16`，`code-simplifier.md:44`）。

### 外部交互
1. Git 交互：读取本地 diff（`git diff --name-only`）。
2. GitHub CLI 交互：读取 PR 视图（`gh pr view`），需要 CLI 可用与鉴权上下文。
3. Task 子代理交互：由主命令向多个审查角色分派任务。

### 运行前提
1. 当前目录是 git 仓库且存在可读 diff。
2. 若需要 PR 上下文，`gh` 命令可执行且具备访问权限。
3. 目标仓库存在足够代码上下文供 agent 分析（仅文档改动时某些审查面可能不激活）。

## 风险、边界与改进建议

### 风险

1. 协议驱动但缺少机器校验
- `review-pr.md` 是文本流程，缺少可执行校验器保证“步骤完整性/参数合法性/agent 引用有效性”。

2. 参数语义存在歧义
- `all parallel` 通过示例表达，而非形式化参数语法；当用户输入混合关键词时，解析一致性依赖模型理解（`review-pr.md:109-113`）。

3. 多 agent 评分体系异构
- 命令汇总层未定义“1-10 / 0-100 / 枚举严重级”的标准映射规则，聚合一致性可能波动。

4. 外部依赖脆弱性
- `gh pr view` 失败时命令中未定义明确降级分支（例如离线/未登录/非 PR 分支）。

5. 文档与协议长期漂移风险
- README、命令文件、agent 文件均描述同一流程，缺少自动一致性检查时容易出现行为与文档不同步。

### 边界

1. 本目录不实现静态分析算法，只定义调用哪些 agent、何时调用、如何汇总。
2. 本目录不直接提交评论到 GitHub，也不运行测试；输出是“评审建议与行动计划”。
3. 本目录不管理插件安装与启用，仅消费插件运行时发现机制。

### 改进建议

1. 增加命令协议 lint 脚本
- 校验 `review-pr.md` 引用的 agent 名是否存在且可解析。
- 校验 `allowed-tools`、`argument-hint` 等关键 frontmatter 字段完整性。

2. 显式定义参数语法
- 在命令文档中加入简明语法（例如 `<aspects...> [parallel]`），减少解析歧义。

3. 统一聚合分级映射
- 在汇总阶段定义固定映射规则（如把 0-100/1-10/枚举统一到 Critical/Important/Suggestion）。

4. 补充失败降级策略
- 明确 `gh pr view` 失败时退化到仅 `git diff` 本地审查，并在总结中注明上下文缺失。

5. 建立 README-命令一致性检查
- 通过轻量脚本对比能力枚举与示例命令，避免文档与协议漂移。
