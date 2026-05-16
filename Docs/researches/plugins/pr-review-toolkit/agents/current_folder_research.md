# plugins/pr-review-toolkit/agents 目录研究（DIR）

## 场景与职责

`plugins/pr-review-toolkit/agents` 是 `pr-review-toolkit` 插件的“专项评审角色层”。该目录不承载可执行业务代码，而是通过 6 份 agent 协议文件定义“在 PR 评审中谁看什么、如何输出、按什么标准打分/分级”。

目录内对象：

- `comment-analyzer.md`：注释准确性与长期可维护性审查（`plugins/pr-review-toolkit/agents/comment-analyzer.md:1-70`）
- `pr-test-analyzer.md`：测试覆盖与测试质量审查（`plugins/pr-review-toolkit/agents/pr-test-analyzer.md:1-69`）
- `silent-failure-hunter.md`：静默失败/错误处理审查（`plugins/pr-review-toolkit/agents/silent-failure-hunter.md:1-130`）
- `type-design-analyzer.md`：类型设计与不变量审查（`plugins/pr-review-toolkit/agents/type-design-analyzer.md:1-110`）
- `code-reviewer.md`：通用代码规范与缺陷审查（`plugins/pr-review-toolkit/agents/code-reviewer.md:1-47`）
- `code-simplifier.md`：通过评审后的代码简化与可维护性优化（`plugins/pr-review-toolkit/agents/code-simplifier.md:1-83`）

在插件链路中的职责位置：

1. 用户通过 `/pr-review-toolkit:review-pr` 进入综合评审流程（`plugins/pr-review-toolkit/commands/review-pr.md:94-113`）。
2. 主命令根据改动类型和参数路由到这些 agent（`plugins/pr-review-toolkit/commands/review-pr.md:20-44`）。
3. agent 返回结构化评审结果，主命令再做聚合总结与行动计划（`plugins/pr-review-toolkit/commands/review-pr.md:57-88`）。

插件层暴露与发现上下文：

- 仓库插件总览将该插件声明为 1 个命令 + 6 个 agents（`plugins/README.md:25`）。
- marketplace 条目将插件源路径注册到 `./plugins/pr-review-toolkit`（`.claude-plugin/marketplace.json:117-125`）。
- 插件自身元数据在 `.claude-plugin/plugin.json`（`plugins/pr-review-toolkit/.claude-plugin/plugin.json:1-9`）。

## 功能点目的

### 1) 以“问题域”拆分评审职责，避免单一审查器过载

`review-pr` 支持 `comments/tests/errors/types/code/simplify/all` 六类关注点（`plugins/pr-review-toolkit/commands/review-pr.md:22-29`），本目录正对应这六类角色。目的不是产出一个笼统结论，而是将“注释、测试、错误处理、类型设计、通用质量、可维护性抛光”分离为可独立调用的审查单元。

### 2) 让每类问题使用不同评估标尺

各 agent 评分/分级机制并不统一，而是按问题性质设计：

- `code-reviewer`：0-100 置信度，且仅报告 `>=80`（`plugins/pr-review-toolkit/agents/code-reviewer.md:22-33`）
- `pr-test-analyzer`：建议项按 1-10 关键度评分（`plugins/pr-review-toolkit/agents/pr-test-analyzer.md:27-48`）
- `type-design-analyzer`：4 维度 1-10 评分（`plugins/pr-review-toolkit/agents/type-design-analyzer.md:24-47,58-69`）
- `silent-failure-hunter`：`CRITICAL/HIGH/MEDIUM` 严重级别（`plugins/pr-review-toolkit/agents/silent-failure-hunter.md:103-109`）

该设计的目标是提高建议可执行性，而不是追求“统一量化分数”。

### 3) 将“发现问题”与“通过后优化”拆为两阶段

`code-simplifier` 在命令说明里被明确定位为“After passing review”的 polish/refine 阶段（`plugins/pr-review-toolkit/commands/review-pr.md:43`），避免在关键缺陷未修复时过早进入“美化重构”。

### 4) 支持串行/并行两种评审编排

命令协议明确：可顺序执行（更易理解）或并行执行（更快）6 个 agent（`plugins/pr-review-toolkit/commands/review-pr.md:47-56,109-113`）。本目录的角色拆分为这种编排方式提供基础。

## 具体技术实现（关键流程/数据结构/协议/命令）

### A. 关键流程（调用方 -> 本目录 -> 聚合输出）

1. 入口命令读取参数 `$ARGUMENTS` 并确定评审面（`plugins/pr-review-toolkit/commands/review-pr.md:11,15-29`）。
2. 命令通过本地上下文识别改动：
- `git diff --name-only`
- `gh pr view`

见 `plugins/pr-review-toolkit/commands/review-pr.md:30-33`。

3. 命令决定适用 agent：
- 总是运行 `code-reviewer`
- 按改动内容条件性运行其他 5 个

见 `plugins/pr-review-toolkit/commands/review-pr.md:38-44`。

4. 命令通过 `Task` 工具调度 agent（`allowed-tools` 包含 `Task`，见 `plugins/pr-review-toolkit/commands/review-pr.md:4`），然后按统一模板聚合结果（`plugins/pr-review-toolkit/commands/review-pr.md:57-88`）。

### B. 数据结构：agent 文件协议

本目录每个文件均采用“YAML frontmatter + 正文系统提示词”结构：

- frontmatter 关键字段：`name`、`description`、`model`（部分含 `color`）
- 正文内容：角色边界、分析流程、输出格式

示例：
- `comment-analyzer.md:1-6`
- `pr-test-analyzer.md:1-6`
- `silent-failure-hunter.md:1-6`
- `type-design-analyzer.md:1-6`
- `code-reviewer.md:1-6`
- `code-simplifier.md:1-36`

### C. 输出协议差异（核心实现细节）

1. `comment-analyzer` 输出分为 `Summary/Critical Issues/Improvement Opportunities/Recommended Removals/Positive Findings`（`plugins/pr-review-toolkit/agents/comment-analyzer.md:48-67`）。
2. `pr-test-analyzer` 输出分为 `Summary/Critical Gaps/Important Improvements/Test Quality Issues/Positive Observations`（`plugins/pr-review-toolkit/agents/pr-test-analyzer.md:49-57`）。
3. `silent-failure-hunter` 单问题输出要求 7 字段（位置、严重级别、影响、隐藏错误类型、修复建议、示例等）（`plugins/pr-review-toolkit/agents/silent-failure-hunter.md:99-110`）。
4. `type-design-analyzer` 固定产出“类型名 + 不变量清单 + 四维评分 + 优化建议”模板（`plugins/pr-review-toolkit/agents/type-design-analyzer.md:50-79`）。
5. `code-reviewer` 明确按高置信过滤并按严重级分组（`plugins/pr-review-toolkit/agents/code-reviewer.md:32-45`）。
6. `code-simplifier` 偏“执行式重构协议”，强调保持功能不变并套用项目规范（`plugins/pr-review-toolkit/agents/code-simplifier.md:42-83`）。

### D. 关键命令与环境依赖

命令层仅声明可用工具，不内嵌具体可执行脚本：

- 允许工具：`Bash`, `Glob`, `Grep`, `Read`, `Task`（`plugins/pr-review-toolkit/commands/review-pr.md:4`）
- 关键命令依赖：`git diff --name-only`, `gh pr view`（`plugins/pr-review-toolkit/commands/review-pr.md:31-33`）

说明该目录属于“提示词协议驱动”的评审系统，而非 shell 脚本驱动系统。

### E. 配置/测试/脚本/文档上下文

1. 配置：
- 插件元数据：`plugins/pr-review-toolkit/.claude-plugin/plugin.json:1-9`
- 仓库 marketplace 注册：`.claude-plugin/marketplace.json:117-125`

2. 测试：
- `plugins/pr-review-toolkit/agents` 目录中无自动化测试文件与测试脚本（目录仅 6 个 `.md`）。

3. 脚本：
- 目标目录无脚本文件；执行依赖 Claude Code 的命令/Task 运行时。

4. 文档：
- 插件 README 对 6 个 agent 的触发语义、工作流和排障有用户级说明（`plugins/pr-review-toolkit/README.md:9-313`）。

## 关键代码路径与文件引用

### 目标目录核心文件

- `plugins/pr-review-toolkit/agents/comment-analyzer.md`
- `plugins/pr-review-toolkit/agents/pr-test-analyzer.md`
- `plugins/pr-review-toolkit/agents/silent-failure-hunter.md`
- `plugins/pr-review-toolkit/agents/type-design-analyzer.md`
- `plugins/pr-review-toolkit/agents/code-reviewer.md`
- `plugins/pr-review-toolkit/agents/code-simplifier.md`

### 上游调用方（谁触发本目录）

1. `plugins/pr-review-toolkit/commands/review-pr.md:20-44`（按改动路由 agent）
2. `plugins/pr-review-toolkit/commands/review-pr.md:45-56`（串行/并行调度策略）
3. `plugins/pr-review-toolkit/README.md:157-180,224-240,289-295`（用户流程中的触发建议）
4. `plugins/README.md:25`（仓库级入口说明）

### 下游依赖（本目录依赖谁）

1. 项目规范文档 `CLAUDE.md`（多个 agent 明确引用其规则）：
- `code-reviewer.md:3,8,16`
- `pr-test-analyzer.md:62`
- `silent-failure-hunter.md:123-129`
- `code-simplifier.md:44-51`

2. 命令行上下文：`git` 与 `gh`（由 `review-pr` 触发，`commands/review-pr.md:31-33`）

3. Claude Code Task 机制（命令 `allowed-tools` 暴露，`commands/review-pr.md:4`）

### 配置与注册路径

- `plugins/pr-review-toolkit/.claude-plugin/plugin.json:1-9`
- `.claude-plugin/marketplace.json:117-125`

## 依赖与外部交互

### 内部依赖

1. 强依赖 `commands/review-pr.md` 的编排语义；没有上游命令调度时，目录内 agent 只作为“可被点名调用”的能力定义存在。
2. 强依赖仓库或项目 `CLAUDE.md` 的规范内容；缺失时审查口径会退化为通用最佳实践。
3. 与 `plugins/pr-review-toolkit/README.md` 用户预期耦合（例如“并行/串行”“评审后再 simplifier”）。

### 外部交互面

1. Git/GitHub CLI 交互：读取工作区改动和 PR 上下文（`git diff --name-only`, `gh pr view`）。
2. 代码库文件系统读取：由各 agent 在分析阶段读取改动文件。
3. 模型运行时交互：
- `code-reviewer` 与 `code-simplifier` 固定 `model: opus`（`code-reviewer.md:4`, `code-simplifier.md:35`）
- 其余 4 个是 `model: inherit`（`comment-analyzer.md:4`, `pr-test-analyzer.md:4`, `silent-failure-hunter.md:4`, `type-design-analyzer.md:4`）

### 边界说明

- 本目录本身不包含可执行脚本、测试、hook、mcp 配置；它仅定义审查行为合同。
- “是否执行修改”在 agent 间并不一致：
- `comment-analyzer` 明确 advisory-only，不直接改代码（`comment-analyzer.md:70`）。
- `code-simplifier` 明确会主动精炼代码（`code-simplifier.md:83`）。

## 风险、边界与改进建议

### 主要风险

1. 口径异构导致汇总困难
- 6 个 agent 使用不同评分体系（百分制、十分制、严重级标签），上游命令虽有统一 summary 模板，但未定义标准化映射规则，跨 agent 排序和自动决策容易不稳定。

2. 规范耦合偏强且带项目假设
- `silent-failure-hunter` 写死了 `logForDebugging/logError/logEvent` 与 `constants/errorIds.ts`（`silent-failure-hunter.md:123-126`），对非该日志栈项目可能产生误报或不适配建议。
- `code-simplifier` 写入了 ES module、`function` 偏好、返回类型等特定约束（`code-simplifier.md:46-51`），跨语言/跨栈通用性有限。

3. 缺少 agent 级回归验证
- 目录只含 prompt 协议，无自动化检测“命令中声明的 agent 是否都存在、输出模板是否被改坏、阈值规则是否漂移”。

4. 模型策略不一致
- 两个高负载 agent 固定 `opus`，其余 `inherit`；在组织层统一模型策略调整时，维护成本和行为一致性风险更高。

5. 指令型文档与可执行性之间存在空隙
- `commands/review-pr.md` 描述了完整流程，但本质是流程指引而非严格执行脚本；效果受运行时模型遵循程度影响。

### 边界

1. 本目录不实现业务逻辑，也不直接触发 git/gh 命令；这些动作在 `commands/review-pr.md` 层定义。
2. 目录输出是“分析建议/审查意见”；最终是否采纳与修改由主流程和开发者决策。
3. 目录无独立测试保障，其质量更多依赖人工评审与实际使用反馈。

### 改进建议

1. 建立统一严重度映射层
- 在 `review-pr` 聚合阶段定义标准映射（例如将 1-10/0-100/标签统一成 Critical/Important/Suggestion），降低多 agent 合并歧义。

2. 为项目特定约束增加可配置开关
- 将 `silent-failure-hunter` 的日志函数与 errorIds 约束改成“可选配置项”而非默认硬编码要求。

3. 增加轻量回归脚本
- 校验 `commands/review-pr.md` 中引用的 6 个 agent 是否都存在；
- 校验关键约束是否存在（如 `code-reviewer` 的 `>=80`）。

4. 收敛模型策略
- 统一采用 `inherit` + 在命令层决定高复杂任务是否升级模型，降低目录内硬编码模型带来的长期维护负担。

5. 在 README 增加“适配性声明”
- 明确哪些规则是 Anthropic 内部项目偏好，哪些是通用最佳实践，减少跨项目误用。
