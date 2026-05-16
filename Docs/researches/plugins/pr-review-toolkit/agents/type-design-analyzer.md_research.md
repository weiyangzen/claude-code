# plugins/pr-review-toolkit/agents/type-design-analyzer.md 研究

## 场景与职责

`type-design-analyzer` 聚焦类型建模质量，核心关注点是“不变量是否被类型系统表达并被边界有效约束”。

流程定位：
- 在 `review-pr` 中对应 `types` 审查面（`plugins/pr-review-toolkit/commands/review-pr.md:25,42,132-136`）。
- README 建议在新增类型、PR 引入数据模型、类型重构时使用（`plugins/pr-review-toolkit/README.md:74-93,179,231`）。

职责边界：
- 评估类型设计强弱与改进方向，不直接负责业务功能评审或性能调优。

## 功能点目的

1. 显性化类型不变量
- 首先识别数据一致性、状态转换、字段关系、业务规则等不变量（`plugins/pr-review-toolkit/agents/type-design-analyzer.md:17-23`）。

2. 提供可比较的四维评分
- Encapsulation / Invariant Expression / Invariant Usefulness / Invariant Enforcement 全部 1-10 量化（`plugins/pr-review-toolkit/agents/type-design-analyzer.md:24-47,58-69`）。

3. 强调“实用而非教条”的改进
- 既强调“illegal states unrepresentable”，也强调复杂度成本与兼容性权衡（`plugins/pr-review-toolkit/agents/type-design-analyzer.md:83-109`）。

4. 统一类型评审输出结构
- 固定模板输出不变量、评分、优点、问题、改进建议，便于 PR 讨论复用（`plugins/pr-review-toolkit/agents/type-design-analyzer.md:50-79`）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) Frontmatter 与配置
- `name: type-design-analyzer`（`plugins/pr-review-toolkit/agents/type-design-analyzer.md:2`）
- `model: inherit`（`plugins/pr-review-toolkit/agents/type-design-analyzer.md:4`）
- `color: pink`（`plugins/pr-review-toolkit/agents/type-design-analyzer.md:5`）

### 2) 分析框架
协议采用 5 步评估：
1. 识别不变量
2. 评估封装性
3. 评估不变量表达清晰度
4. 评估不变量的业务有效性
5. 评估不变量执行力度（构造/变更路径）

证据：`plugins/pr-review-toolkit/agents/type-design-analyzer.md:15-47`。

### 3) 输出模板协议
标准输出包含：
- `Type` 标题
- `Invariants Identified`
- `Ratings`（四维）
- `Strengths`
- `Concerns`
- `Recommended Improvements`

证据：`plugins/pr-review-toolkit/agents/type-design-analyzer.md:50-79`。

### 4) 隐含数据结构
- `TypeInvariant`：`{name, description, enforcement_point}`。
- `TypeScoreCard`：`{encapsulation, expression, usefulness, enforcement}`。
- `ImprovementPlan`：`{change, complexity_cost, breakage_risk, expected_gain}`。

### 5) 命令与调用关系
- `review-pr` 在类型改动时激活该 agent，并与其它审查项并行/串行组合（`plugins/pr-review-toolkit/commands/review-pr.md:35-56`）。
- README 的“After adding types”触发建议与命令路由保持一致（`plugins/pr-review-toolkit/README.md:179`）。

### 6) 配置、测试、脚本、文档上下文
- 配置：agent frontmatter；插件级配置 `plugin.json`（`plugins/pr-review-toolkit/.claude-plugin/plugin.json:1-9`）。
- 测试：无“类型评审结论一致性”回归用例。
- 脚本：无专用脚本，属于纯提示协议角色。
- 文档：README 提供触发词与评分预期（`plugins/pr-review-toolkit/README.md:74-93,205,231`）。

## 关键代码路径与文件引用

- Agent 定义：`plugins/pr-review-toolkit/agents/type-design-analyzer.md:1-110`
- 命令映射与编排：`plugins/pr-review-toolkit/commands/review-pr.md:25,42,45-56,132-136,167-170`
- 用户文档：`plugins/pr-review-toolkit/README.md:74-93,179,205,231`
- 插件元数据：`plugins/pr-review-toolkit/.claude-plugin/plugin.json:1-9`
- 插件索引：`plugins/README.md:25`
- 市场注册：`.claude-plugin/marketplace.json:117-125`

## 依赖与外部交互

1. 内部依赖
- 依赖类型定义与相关业务上下文可读，否则难以识别真实不变量。
- 依赖上游命令准确识别“类型相关改动”。

2. 外部交互
- 主要是代码语义读取与推理，不直接依赖外部 API。
- 在 PR 场景下，受 `git/gh` 上下文判定间接影响。

3. 运行前提
- 对静态类型语言（TS/Rust/Java 等）效果更直接；动态语言需要额外约束信息支持。
- 需要开发者提供或保留业务规则上下文，避免“仅凭结构打分”。

## 风险、边界与改进建议

1. 风险
- 评分主观性：四维 1-10 在缺乏标注样例时，跨评审者一致性有限。
- 语言差异风险：同一框架用于动态语言时，`compile-time guarantees` 建议可能不适配。
- 聚合异构：与其它 agent 的 0-100/标签输出合并时需映射策略。

2. 边界
- 不负责执行迁移重构或自动修复类型设计，只给审查意见。
- 不替代领域建模讨论；只能基于现有代码上下文提出改进。

3. 改进建议
- 增加评分锚点示例（每个分数区间对应典型特征）。
- 在输出中加入“改造成本等级”（低/中/高），辅助团队排期。
- 在命令聚合层统一分数映射，降低多 agent 结果比较难度。
