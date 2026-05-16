# plugins/pr-review-toolkit/agents/code-simplifier.md 研究

## 场景与职责

`code-simplifier` 是该插件中的“评审后优化”角色，目标不是找 bug，而是在功能不变前提下提升代码可读性、一致性与可维护性。

在流程中的位置：
- `review-pr` 将其放在“After passing review”阶段（`plugins/pr-review-toolkit/commands/review-pr.md:43`）。
- README 也将它置于“通过评审后”步骤（`plugins/pr-review-toolkit/README.md:128-129,234-236,294`）。

职责边界：
- 相比多数 advisory 型审查 agent，它是执行式精炼角色，允许直接提出并执行重构方向。
- 仅聚焦近期改动，避免无界重写（`plugins/pr-review-toolkit/agents/code-simplifier.md:72`）。

## 功能点目的

1. 建立“先正确，再简化”的顺序
- 与 `code-reviewer`/专项审查分层，减少在缺陷未收敛前过早做样式重构。

2. 在不改行为的条件下降低复杂度
- 第一原则是“Preserve Functionality”（`plugins/pr-review-toolkit/agents/code-simplifier.md:42`）。

3. 强化项目一致性
- 明确要求跟随 `CLAUDE.md` 约束（ESM、函数声明偏好、React Props、错误处理风格等）（`plugins/pr-review-toolkit/agents/code-simplifier.md:44-51`）。

4. 避免“短但难读”的伪优化
- 明确禁止嵌套三元、反对过度紧凑表达，强调清晰优先（`plugins/pr-review-toolkit/agents/code-simplifier.md:60-71`）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) Frontmatter 与配置
- `name: code-simplifier`（`plugins/pr-review-toolkit/agents/code-simplifier.md:2`）
- `model: opus`（`plugins/pr-review-toolkit/agents/code-simplifier.md:35`）
- `description` 强调“写完代码后自动触发精炼”语义（`plugins/pr-review-toolkit/agents/code-simplifier.md:3`）

该配置体现其定位为高质量改写器，而非单纯评论器。

### 2) 五类改进约束
协议按五个目标组织：
- 功能不变
- 对齐项目标准
- 提升清晰度
- 避免过度简化
- 控制范围（近期改动）

证据：`plugins/pr-review-toolkit/agents/code-simplifier.md:42-73`。

### 3) 执行流程
定义了 6 步 refinement process：
1. 定位近期改动
2. 识别可简化机会
3. 套用项目标准
4. 校验行为不变
5. 验证可维护性提升
6. 仅记录重要改动

证据：`plugins/pr-review-toolkit/agents/code-simplifier.md:74-82`。

### 4) 关键协议数据结构（隐含）
- `TouchedScope`：当前会话/近期改动集合。
- `RefinementCandidate`：`{location, complexity_smell, simplification, behavior_risk}`。
- `AcceptedRefinement`：满足功能等价 + 可读性收益。

### 5) 命令与调用关系
- 上游由 `review-pr` 选择串行或并行调度（`plugins/pr-review-toolkit/commands/review-pr.md:45-56`）。
- 使用建议在 README 的工作流与 Tips 中重复出现（`plugins/pr-review-toolkit/README.md:116-138,234-236,289-295`）。

### 6) 配置、测试、脚本、文档上下文
- 配置：frontmatter + 插件 manifest（`plugins/pr-review-toolkit/.claude-plugin/plugin.json:1-9`）。
- 测试：目录中未见针对“功能等价”验证的自动化脚本，行为保持主要依赖提示协议。
- 脚本：无专属脚本；通过命令层 Task 触发。
- 文档：README 说明其“保留功能”与使用时机（`plugins/pr-review-toolkit/README.md:116-138`）。

## 关键代码路径与文件引用

- Agent 定义：`plugins/pr-review-toolkit/agents/code-simplifier.md:1-83`
- 命令调度与阶段定位：`plugins/pr-review-toolkit/commands/review-pr.md:38-44,45-56,105-107,142-146`
- 用户文档：`plugins/pr-review-toolkit/README.md:116-138,209,234-236,294`
- 插件元数据：`plugins/pr-review-toolkit/.claude-plugin/plugin.json:1-9`
- 仓库插件索引：`plugins/README.md:25`
- 市场注册：`.claude-plugin/marketplace.json:117-125`

## 依赖与外部交互

1. 内部依赖
- 依赖上游先完成缺陷审查，再进入可维护性优化阶段。
- 强依赖 `CLAUDE.md` 风格规则（`plugins/pr-review-toolkit/agents/code-simplifier.md:44-51`）。

2. 外部交互
- 通过 Task 与运行时交互，不直接定义 shell 命令。
- 间接依赖 git 变更范围（由调用方确定“近期改动”）。

3. 运行前提
- 需要明确待精炼代码范围；若范围不清晰，容易误改无关代码。
- 理想情况下应有测试或可执行验证兜底，以证明“功能不变”。

## 风险、边界与改进建议

1. 风险
- 风格硬编码风险：ESM、`function` 偏好、React 模式等规则对非 JS/TS 项目不一定适配。
- 功能等价不可自动证明：当前协议强调“不改行为”，但缺少内建验证步骤。
- 自动触发语义可能与用户意图冲突：在赶工修 bug 时，过早重构会增加 review 负担。

2. 边界
- 仅建议精炼近期改动，不是全仓重构器。
- 目标是“结构更清晰”，不是“代码行数最少”。

3. 改进建议
- 为 `code-simplifier` 增加 `mode` 约定：`advisory`（只给建议）/`apply`（执行改写）。
- 在输出中强制包含“行为保持依据”（测试、逻辑等价说明、未触及接口声明）。
- 把项目特定规则改为可配置条目，减少跨项目误报。
