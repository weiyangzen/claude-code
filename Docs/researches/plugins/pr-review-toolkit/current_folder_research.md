# plugins/pr-review-toolkit 目录研究（DIR）

## 场景与职责

`plugins/pr-review-toolkit` 是一个“PR 评审能力聚合插件”，核心入口是命令 `/pr-review-toolkit:review-pr`，核心目标是在创建/更新 PR 前后，把不同维度的质量检查拆给 6 个专门 agent，形成可组合的审查流程。

目录与职责分层：
1. `commands/review-pr.md`：定义评审编排协议（范围判定、工具调用、并行/串行、结果聚合、行动建议）。
2. `agents/*.md`：6 个专项审查 agent 的角色边界、评审标准、输出格式。
3. `.claude-plugin/plugin.json`：插件元数据（名称、版本、作者）。
4. `README.md`：用户侧使用说明、触发语句、工作流建议、故障排查。

插件上下文定位：
- 在 marketplace 中注册为 `pr-review-toolkit`，源目录为 `./plugins/pr-review-toolkit`：`.claude-plugin/marketplace.json:117-125`
- 在插件总览中作为“PR review toolkit”暴露：`plugins/README.md:25`
- 根 README 将 `plugins/README.md` 作为插件入口文档：`README.md:48-50`

该目录没有传统可执行源码（`.ts/.js/.py`）、也没有目录内脚本或自动化测试；行为主要由 Markdown 协议（frontmatter + 指令文本）驱动。

## 功能点目的

### 1) 提供“按方面选择”的 PR 评审入口

`review-pr` 支持 `comments/tests/errors/types/code/simplify/all`，目的不是单一评分，而是按问题类别路由到最合适的子 agent：`plugins/pr-review-toolkit/commands/review-pr.md:20-29`。

### 2) 通过变更感知提升评审相关性

命令协议要求先看 `git diff --name-only`，并尝试 `gh pr view` 判断 PR 上下文，再根据文件类型和改动内容决定是否启用某些审查面：`plugins/pr-review-toolkit/commands/review-pr.md:30-43`。

### 3) 把“通用审查 + 专项审查 + 简化重构”串成完整闭环

职责分配：
- 通用质量底盘：`code-reviewer`
- 领域专项：`comment-analyzer` / `pr-test-analyzer` / `silent-failure-hunter` / `type-design-analyzer`
- 收尾优化：`code-simplifier`

命令中明确 `code-simplifier` 是“通过审查后的 polish/refine”角色：`plugins/pr-review-toolkit/commands/review-pr.md:38-44`。

### 4) 支持串行与并行两种执行策略

- 串行：利于逐项处理反馈。
- 并行：压缩总体评审时长。

对应协议：`plugins/pr-review-toolkit/commands/review-pr.md:45-56,109-113`。

### 5) 统一结果出口，降低行动成本

命令定义统一汇总模板（Critical / Important / Suggestions / Strengths + Action Plan），让多 agent 输出可直接转成修复清单：`plugins/pr-review-toolkit/commands/review-pr.md:57-88`。

## 具体技术实现（关键流程/数据结构/协议/命令）

### A. 关键流程（端到端）

1. 插件发现与命令暴露
- marketplace 项将插件源目录映射到运行时：`.claude-plugin/marketplace.json:117-125`
- 目录结构遵循插件约定（`.claude-plugin/`、`commands/`、`agents/`、`README.md`）：`plugins/README.md:49-61`

2. 用户触发命令
- 用户执行 `/pr-review-toolkit:review-pr [review-aspects]`。
- 命令 frontmatter 提供 `description`、`argument-hint`、`allowed-tools`；正文通过 `$ARGUMENTS` 接收参数：`plugins/pr-review-toolkit/commands/review-pr.md:1-12`

3. 范围判定与评审面选择
- 先取变更文件与 PR 状态：`git diff --name-only`、`gh pr view`：`plugins/pr-review-toolkit/commands/review-pr.md:30-33`
- 再按变更类型映射可用审查面：`plugins/pr-review-toolkit/commands/review-pr.md:35-44`

4. 子 agent 调度
- 命令允许 `Task` 工具，意图是由主命令编排并启动各 agent：`plugins/pr-review-toolkit/commands/review-pr.md:4`
- agent 名与职责定义在 `agents/*.md` frontmatter：
  - `comment-analyzer`：`plugins/pr-review-toolkit/agents/comment-analyzer.md:1-6`
  - `pr-test-analyzer`：`plugins/pr-review-toolkit/agents/pr-test-analyzer.md:1-6`
  - `silent-failure-hunter`：`plugins/pr-review-toolkit/agents/silent-failure-hunter.md:1-6`
  - `type-design-analyzer`：`plugins/pr-review-toolkit/agents/type-design-analyzer.md:1-6`
  - `code-reviewer`：`plugins/pr-review-toolkit/agents/code-reviewer.md:1-6`
  - `code-simplifier`：`plugins/pr-review-toolkit/agents/code-simplifier.md:1-36`

5. 结果聚合与行动计划输出
- 将多 agent 结果统一归档到三类问题 + 强项 + 修复顺序：`plugins/pr-review-toolkit/commands/review-pr.md:57-88`

### B. 数据结构与协议

1. 插件元数据（JSON）
- 字段：`name/version/description/author`：`plugins/pr-review-toolkit/.claude-plugin/plugin.json:1-9`
- 用于插件身份声明与展示。

2. 命令协议（Markdown + YAML frontmatter）
- `description`：命令功能说明
- `argument-hint`：参数提示（`[review-aspects]`）
- `allowed-tools`：`["Bash", "Glob", "Grep", "Read", "Task"]`
- `$ARGUMENTS`：用户输入参数注入
证据：`plugins/pr-review-toolkit/commands/review-pr.md:1-5,11`

3. 评审面枚举（隐式数据模型）
- `comments/tests/errors/types/code/simplify/all`
- 兼容“all + parallel”运行形态
证据：`plugins/pr-review-toolkit/commands/review-pr.md:20-29,109-113`

4. 统一汇总结构（输出合同）
- 一级分类：`Critical Issues / Important Issues / Suggestions / Strengths / Recommended Action`
证据：`plugins/pr-review-toolkit/commands/review-pr.md:67-88`

5. 各 agent 的评分/分级协议
- `pr-test-analyzer`：关键建议按 1-10 评分并分层：`plugins/pr-review-toolkit/agents/pr-test-analyzer.md:27-48`
- `type-design-analyzer`：4 维度 1-10 评分：`plugins/pr-review-toolkit/agents/type-design-analyzer.md:24-47,58-69`
- `code-reviewer`：0-100 置信度并仅报告 >=80：`plugins/pr-review-toolkit/agents/code-reviewer.md:22-33`
- `silent-failure-hunter`：`CRITICAL/HIGH/MEDIUM` 严重级别：`plugins/pr-review-toolkit/agents/silent-failure-hunter.md:103-109`

### C. 关键命令与标准检查点

1. Git/GitHub 命令依赖（由命令协议要求）
- `git diff --name-only`：识别改动面
- `gh pr view`：检查 PR 上下文
证据：`plugins/pr-review-toolkit/commands/review-pr.md:31-33`

2. 项目规范依赖
- `code-reviewer`、`pr-test-analyzer`、`silent-failure-hunter`、`code-simplifier` 多处提及按 `CLAUDE.md` 约束审查：
  - `plugins/pr-review-toolkit/agents/code-reviewer.md:3,8,16`
  - `plugins/pr-review-toolkit/agents/pr-test-analyzer.md:62`
  - `plugins/pr-review-toolkit/agents/silent-failure-hunter.md:123-129`
  - `plugins/pr-review-toolkit/agents/code-simplifier.md:44`

3. 命令与文档一致性
- README 对命令示例、并行执行、工作流顺序有详细说明：`plugins/pr-review-toolkit/README.md:90-113,241-253,289-295`

## 关键代码路径与文件引用

### 目录内核心路径

- 插件元数据：`plugins/pr-review-toolkit/.claude-plugin/plugin.json:1-9`
- 主命令协议：`plugins/pr-review-toolkit/commands/review-pr.md:1-189`
- 六个 agent 定义：
  - `plugins/pr-review-toolkit/agents/comment-analyzer.md:1-70`
  - `plugins/pr-review-toolkit/agents/pr-test-analyzer.md:1-69`
  - `plugins/pr-review-toolkit/agents/silent-failure-hunter.md:1-130`
  - `plugins/pr-review-toolkit/agents/type-design-analyzer.md:1-110`
  - `plugins/pr-review-toolkit/agents/code-reviewer.md:1-47`
  - `plugins/pr-review-toolkit/agents/code-simplifier.md:1-83`
- 用户文档：`plugins/pr-review-toolkit/README.md:1-313`

### 调用方（上游）

1. marketplace 插件发现机制：`.claude-plugin/marketplace.json:117-125`
2. 插件总览入口：`plugins/README.md:25`
3. 用户命令触发：`/pr-review-toolkit:review-pr`（示例见 `plugins/pr-review-toolkit/commands/review-pr.md:94-113`）

### 被调用方（下游）

1. Task 子代理系统（运行 6 个 agent）：`plugins/pr-review-toolkit/commands/review-pr.md:4`
2. Shell 命令环境（git/gh）：`plugins/pr-review-toolkit/commands/review-pr.md:31-33`
3. 仓库代码上下文（按 diff 与文件类型选择评审）

### 配置、测试、脚本、文档覆盖结论

1. 配置
- 插件级：`.claude-plugin/plugin.json`
- 命令级：`commands/review-pr.md` frontmatter
- agent 级：`agents/*.md` frontmatter（`name/description/model/color`）

2. 测试
- 目录内无自动化测试文件（无 `test/spec/__tests__` 目录或测试脚本）。

3. 脚本
- 目录内无 `scripts/` 或 `.sh/.py/.js/.ts` 执行脚本。

4. 文档
- 目录内 `README.md` 覆盖能力、用法、工作流、排障：`plugins/pr-review-toolkit/README.md:1-313`
- 全局索引文档 `plugins/README.md` 提供对外入口：`plugins/README.md:25`

## 依赖与外部交互

### 运行时依赖

1. Claude Code 插件运行时（命令与 agents 的自动发现/执行）。
2. `Task` 子代理能力（用于拆分专项审查）。
3. Git 环境（`git diff`）。
4. GitHub CLI 与登录上下文（`gh pr view`）。
5. 目标仓库上下文（代码 diff、文件类型、项目规范）。

### 外部交互面

1. GitHub 交互
- 通过 `gh pr view` 读取 PR 元信息：`plugins/pr-review-toolkit/commands/review-pr.md:32`

2. Shell 交互
- 通过 Bash 执行本地 git/gh 命令：`plugins/pr-review-toolkit/commands/review-pr.md:4,31-33`

3. 规范交互
- 通过 `CLAUDE.md`（若存在）注入项目级规则到多个 agent 的判断逻辑。

### 依赖边界

- 该插件本身不直接实现静态分析器或测试运行器；它是“评审编排层”，把分析责任委托给子 agent 的提示协议与当前仓库上下文。
- 因此能力上限取决于：agent prompt 质量、上下文完整性、git/gh 可用性。

## 风险、边界与改进建议

### 风险与边界

1. 规范依赖偏强但非强制存在
- 多个 agent 要求参考 `CLAUDE.md`，但当前仓库未必存在该文件；在无规范文件时，审查口径会退化为通用经验判断。

2. Prompt 协议与真实执行存在漂移风险
- `review-pr.md` 描述了“判定适用审查 + 串/并行调度 + 汇总输出”，但缺少可执行脚本化校验，后续改文案容易造成行为偏差。

3. 文档作者信息不一致
- 插件 manifest 作者是 `Daisy`：`plugins/pr-review-toolkit/.claude-plugin/plugin.json:6-7`
- marketplace 作者是 `Anthropic`：`.claude-plugin/marketplace.json:121-123`
- 影响：发布审计或归属追踪可能出现歧义。

4. 项目特定假设可能导致泛化误报
- `silent-failure-hunter` 强绑定某些日志函数与 `constants/errorIds.ts`：`plugins/pr-review-toolkit/agents/silent-failure-hunter.md:41-42,123-126`
- `code-simplifier` 假设 ES module / function 风格等约束：`plugins/pr-review-toolkit/agents/code-simplifier.md:46-51`
- 当目标仓库规范不同，可能出现“正确但被判不合规”的误报。

5. 工具权限面较宽
- 命令允许 `Bash/Glob/Grep/Read/Task`；在缺少更细粒度约束时，执行面较依赖模型自律：`plugins/pr-review-toolkit/commands/review-pr.md:4`

6. 无目录内测试与脚本回归
- 目录内没有自动化验证手段来确保 frontmatter、agent 名称引用、输出模板稳定。

### 改进建议

1. 增加轻量一致性检查脚本
- 校验项：
  - `commands/review-pr.md` 中引用的 6 个 agent 名是否都存在对应文件
  - frontmatter 字段完整性（`description/allowed-tools`）
  - README 示例命令是否与命令文件保持一致

2. 将“项目特定规则”参数化
- 把 `silent-failure-hunter` 中日志函数名、error ID 路径改成“可覆盖变量/占位说明”，降低跨项目误报。

3. 对齐作者元信息
- 统一 `.claude-plugin/plugin.json` 与 marketplace 的作者字段，减少治理歧义。

4. 在命令中补充失败分支约定
- 明确 `gh pr view` 失败、无 diff、无匹配评审面时的降级输出，提升鲁棒性。

5. 建议增加结果结构化字段
- 在汇总里加入 `reviewed_files`、`activated_agents`、`skipped_aspects`，便于后续自动化消费与审计。
