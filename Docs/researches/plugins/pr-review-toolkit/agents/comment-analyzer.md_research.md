# plugins/pr-review-toolkit/agents/comment-analyzer.md 研究

## 场景与职责

`comment-analyzer` 负责注释质量治理，核心任务是避免“注释腐烂（comment rot）”带来的长期维护负担。

流程定位：
- 在 `review-pr` 中对应 `comments` 审查面（`plugins/pr-review-toolkit/commands/review-pr.md:22,40,117-121`）。
- README 将其用于“文档新增后、PR 提交前、注释变更复核”场景（`plugins/pr-review-toolkit/README.md:11-30,230`）。

职责边界：
- 该 agent 明确是 advisory-only，不直接修改代码或注释（`plugins/pr-review-toolkit/agents/comment-analyzer.md:70`）。

## 功能点目的

1. 验证注释与代码事实一致
- 优先检查签名、行为、边界条件、复杂度描述是否与实现匹配（`plugins/pr-review-toolkit/agents/comment-analyzer.md:14-20`）。

2. 识别低价值注释与潜在技术债
- 标记“重复代码表面含义”的注释、过时说明、模糊措辞（`plugins/pr-review-toolkit/agents/comment-analyzer.md:28-40`）。

3. 保证注释具备长期可维护价值
- 关注“why”而非“what”，避免临时态描述（`plugins/pr-review-toolkit/agents/comment-analyzer.md:29-34`）。

4. 输出可执行修改建议而非泛泛评价
- 输出结构要求具体到位置、问题与改写建议（`plugins/pr-review-toolkit/agents/comment-analyzer.md:48-67`）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) Frontmatter 与配置
- `name: comment-analyzer`（`plugins/pr-review-toolkit/agents/comment-analyzer.md:2`）
- `model: inherit`（`plugins/pr-review-toolkit/agents/comment-analyzer.md:4`）
- `color: green`（`plugins/pr-review-toolkit/agents/comment-analyzer.md:5`）

`inherit` 表示模型策略由上层会话决定，便于统一调度成本与能力。

### 2) 五步分析框架
协议给出 5 维检查：
1. Factual Accuracy
2. Completeness
3. Long-term Value
4. Misleading Elements
5. Improvement Suggestions

证据：`plugins/pr-review-toolkit/agents/comment-analyzer.md:12-47`。

### 3) 输出协议结构
固定输出区块：
- `Summary`
- `Critical Issues`
- `Improvement Opportunities`
- `Recommended Removals`
- `Positive Findings`

并要求每条包含位置与建议（`plugins/pr-review-toolkit/agents/comment-analyzer.md:48-67`）。

### 4) 隐含数据结构
- `CommentClaim`：注释中的事实性声明。
- `VerificationResult`：`{location, claim_status, mismatch_reason}`。
- `Recommendation`：`{type: fix|enhance|remove, rationale, rewrite}`。

### 5) 调用与命令关系
- 由 `review-pr` 在注释/文档改动场景触发（`plugins/pr-review-toolkit/commands/review-pr.md:40`）。
- 也可在自然语言触发词下独立调用（`plugins/pr-review-toolkit/README.md:25-30,153-155`）。

### 6) 配置、测试、脚本、文档上下文
- 配置：agent frontmatter + 插件 manifest（`plugins/pr-review-toolkit/.claude-plugin/plugin.json:1-9`）。
- 测试：无“注释正确性评估”自动回归样例。
- 脚本：无配套脚本；靠 Task 调度 + 仓库上下文读取。
- 文档：README 明确其触发语句与使用窗口（`plugins/pr-review-toolkit/README.md:11-31,176-178,229-231,293`）。

## 关键代码路径与文件引用

- Agent 定义：`plugins/pr-review-toolkit/agents/comment-analyzer.md:1-70`
- 命令路由与说明：`plugins/pr-review-toolkit/commands/review-pr.md:20-44,117-121`
- 用户文档：`plugins/pr-review-toolkit/README.md:11-31,153-155,199,229-231,293`
- 插件元数据：`plugins/pr-review-toolkit/.claude-plugin/plugin.json:1-9`
- 仓库索引：`plugins/README.md:25`
- 注册入口：`.claude-plugin/marketplace.json:117-125`

## 依赖与外部交互

1. 内部依赖
- 依赖可访问到“注释与对应代码实现”的上下文，否则无法完成事实核验。
- 依赖主流程将输出转化为实际改动（本 agent 不直接修改）。

2. 外部交互
- 主要是仓库文件读取与语义比对，不强依赖外部网络服务。
- 在 `review-pr` 场景下，间接受到 `git diff/gh pr view` 上下文筛选影响。

3. 运行前提
- 需要明确分析范围（最近改动、指定文件或 PR）。
- 对多语言仓库，注释规范差异会影响判定一致性。

## 风险、边界与改进建议

1. 风险
- 缺少统一严重度量化：当前是结构化分区而非数值评分，跨 agent 聚合时排序较弱。
- 语义误判风险：复杂业务注释中的领域术语可能被误判为“模糊或冗余”。
- advisory-only 落地断层：若主流程未跟进修订，问题会持续累积。

2. 边界
- 只做分析与建议，不直接实施修改。
- 重点是维护期价值，不是“注释越多越好”。

3. 改进建议
- 增加轻量优先级标签（P0/P1/P2）以便与其它 agent 结果统一排序。
- 在输出中加入 `scope_covered`（扫描文件与区块）减少“漏扫/误扫”争议。
- 为常见注释模式建立示例库（好/坏对照）提升跨团队一致性。
