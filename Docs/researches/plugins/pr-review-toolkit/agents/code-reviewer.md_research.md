# plugins/pr-review-toolkit/agents/code-reviewer.md 研究

## 场景与职责

`code-reviewer` 是 `pr-review-toolkit` 中的通用质量审查代理，定位为“默认总是执行”的基线审查器，用于在 PR 评审阶段先兜住规范违例与高风险缺陷。

在编排链路中的位置：
- `review-pr` 将其标记为 Always applicable（`plugins/pr-review-toolkit/commands/review-pr.md:38`）。
- README 将其放在“写完代码后 / 提交前 / PR 前”使用（`plugins/pr-review-toolkit/README.md:95-114,224-233`）。

职责边界：
- 主要输出高置信问题清单与修复建议，不承担最终合并决策。
- 默认审查范围是未暂存改动（`git diff`），可由调用方指定其它范围（`plugins/pr-review-toolkit/agents/code-reviewer.md:12`）。

## 功能点目的

1. 作为综合评审的统一入口
- 在六个专项 agent 中，`code-reviewer` 是唯一被命令层声明“总是适用”的角色（`plugins/pr-review-toolkit/commands/review-pr.md:38`）。

2. 通过高置信阈值降低噪声
- 定义 0-100 置信度并强制仅报告 `>=80`，避免将大量边缘建议混入阻断项（`plugins/pr-review-toolkit/agents/code-reviewer.md:22-33`）。

3. 把“规范合规 + 真实 bug + 关键质量问题”放在同一检查面
- 核心职责覆盖 CLAUDE.md 合规、功能缺陷、关键质量问题（`plugins/pr-review-toolkit/agents/code-reviewer.md:16-21`）。

4. 为后续聚合输出提供可排序结构
- 输出要求包含严重度分组、文件行号、依据与修复建议（`plugins/pr-review-toolkit/agents/code-reviewer.md:34-45`），便于上游在 `PR Review Summary` 中汇总（`plugins/pr-review-toolkit/commands/review-pr.md:67-88`）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) Frontmatter 与运行配置
- `name: code-reviewer`（`plugins/pr-review-toolkit/agents/code-reviewer.md:2`）
- `model: opus`（`plugins/pr-review-toolkit/agents/code-reviewer.md:4`）
- `color: green`（`plugins/pr-review-toolkit/agents/code-reviewer.md:5`）
- `description` 明确“提交前/PR 前主动触发”与默认 diff 范围（`plugins/pr-review-toolkit/agents/code-reviewer.md:3`）

这意味着它是高负载质量门角色，模型选择偏稳定输出，而非最低成本。

### 2) 审查流程协议
隐含流程可拆为：
1. 确认审查范围（默认 unstaged diff，或调用方给定路径）。
2. 按三大责任域扫描：规范、bug、代码质量。
3. 对候选问题做 0-100 置信评分。
4. 仅保留 `>=80` 条目。
5. 输出按 `Critical (90-100)` / `Important (80-89)` 分组结果。

对应证据：`plugins/pr-review-toolkit/agents/code-reviewer.md:10-45`。

### 3) 关键数据结构（协议层）
可抽象为：
- `ReviewScope`：`git diff` 默认范围 + 可选显式文件集。
- `Finding`：`{confidence, severity_bucket, file, line, rule_or_bug_reason, fix_suggestion}`。
- `SeverityBucket`：`Critical` 或 `Important`（由分数映射）。

该结构与 `review-pr` 聚合模板相容（`plugins/pr-review-toolkit/commands/review-pr.md:69-88`）。

### 4) 调用/命令关系
- 上游命令通过 `Task` 调度 agent（`plugins/pr-review-toolkit/commands/review-pr.md:4,45-56`）。
- 命令会先用 `git diff --name-only`、`gh pr view`确定上下文（`plugins/pr-review-toolkit/commands/review-pr.md:31-33`），再触发本 agent。

### 5) 配置、测试、脚本、文档上下文
- 配置：插件元数据来自 `plugins/pr-review-toolkit/.claude-plugin/plugin.json:1-9`；agent 自身配置在 frontmatter。
- 测试：本插件目录无针对该 agent 的自动化契约测试（无测试脚本/测试目录资产）。
- 脚本：`code-reviewer` 本身无可执行脚本，依赖运行时与命令层调度。
- 文档：README 的 agent 说明与使用时机为用户预期来源（`plugins/pr-review-toolkit/README.md:95-114,176-177,224-233,290`）。

## 关键代码路径与文件引用

- Agent 定义：`plugins/pr-review-toolkit/agents/code-reviewer.md:1-47`
- 主命令编排：`plugins/pr-review-toolkit/commands/review-pr.md:20-44,45-56,67-88,117-140`
- 插件用户文档：`plugins/pr-review-toolkit/README.md:95-114,176-177,199-209,224-233,290`
- 插件元数据：`plugins/pr-review-toolkit/.claude-plugin/plugin.json:1-9`
- 仓库插件索引：`plugins/README.md:25`
- 插件市场注册：`.claude-plugin/marketplace.json:117-125`
- 命令触发 agent 的平台约定：`plugins/plugin-dev/skills/command-development/references/plugin-features-reference.md:334-356`

## 依赖与外部交互

1. 内部依赖
- 依赖 `review-pr` 的激活策略；若不经命令层，本 agent 需要被显式点名调用。
- 依赖项目规范源（文中多次引用 `CLAUDE.md`，见 `plugins/pr-review-toolkit/agents/code-reviewer.md:3,8,16`）。

2. 外部交互
- Git/PR 上下文由上游命令获取（`git diff --name-only`, `gh pr view`）。
- 与 Claude Code Task 机制交互，用于多 agent 协同执行（`plugins/pr-review-toolkit/commands/review-pr.md:4`）。

3. 运行前提
- 当前目录可读取 git 变更。
- 若需要 PR 视图，`gh` 可用且已鉴权。
- 代码库存在可落地的项目规则文档；否则审查会退化为通用最佳实践判断。

## 风险、边界与改进建议

1. 风险
- 规则依赖漂移：agent 强调 `CLAUDE.md`，但当前仓库未发现该文件，可能导致“规范判断口径不一致”。
- 阈值过高的漏报风险：`>=80` 能降噪，但会压制一部分中等风险问题。
- 模型硬编码成本：固定 `model: opus` 在大规模并行评审时有资源成本压力。

2. 边界
- 该 agent 以“发现问题 + 建议修复”为主，不直接承诺自动改码。
- 默认范围是近期改动，不等同于全仓深审。

3. 改进建议
- 增加范围回显字段：输出首段固定声明 `review_scope`（unstaged/staged/specified files）。
- 在聚合层定义阈值外提示：例如把 70-79 作为“观察项”可选输出，避免完全丢失中风险信号。
- 增加轻量契约检查：校验 `>=80` 规则、严重级分组、输出字段完整性。
