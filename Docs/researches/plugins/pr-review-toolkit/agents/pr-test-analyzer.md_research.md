# plugins/pr-review-toolkit/agents/pr-test-analyzer.md 研究

## 场景与职责

`pr-test-analyzer` 是 PR 测试覆盖审查角色，聚焦“行为覆盖是否足以防回归”，而不是追求机械化 100% 行覆盖率。

流程定位：
- 在 `review-pr` 中对应 `tests` 评审面（`plugins/pr-review-toolkit/commands/review-pr.md:23,39,122-126`）。
- README 将其定位为“PR 创建后/新增功能后”的覆盖核验工具（`plugins/pr-review-toolkit/README.md:32-51,229,292`）。

职责边界：
- 以分析与建议为主，不直接执行测试命令或改写测试代码。

## 功能点目的

1. 让评审聚焦真实回归风险
- 明确“behavioral coverage > line coverage”（`plugins/pr-review-toolkit/agents/pr-test-analyzer.md:12`）。

2. 优先补齐关键路径缺口
- 指向错误处理、边界条件、业务分支、负例、并发/异步行为缺失（`plugins/pr-review-toolkit/agents/pr-test-analyzer.md:14-20`）。

3. 同时评估测试质量而非只看有没有测
- 检查行为导向、抗重构脆弱性、DAMP 清晰度（`plugins/pr-review-toolkit/agents/pr-test-analyzer.md:21-26`）。

4. 给出可排序、可落地的建议
- 每项建议要求关键度 1-10 + 能防止的具体回归说明（`plugins/pr-review-toolkit/agents/pr-test-analyzer.md:27-31`）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) Frontmatter 与配置
- `name: pr-test-analyzer`（`plugins/pr-review-toolkit/agents/pr-test-analyzer.md:2`）
- `model: inherit`（`plugins/pr-review-toolkit/agents/pr-test-analyzer.md:4`）
- `color: cyan`（`plugins/pr-review-toolkit/agents/pr-test-analyzer.md:5`）

### 2) 分析流程协议
六步流程：
1. 理解 PR 新增/修改功能
2. 映射现有测试覆盖
3. 找出生产风险关键路径
4. 识别实现耦合过深的测试
5. 识别缺失负例和异常场景
6. 评估集成点覆盖

证据：`plugins/pr-review-toolkit/agents/pr-test-analyzer.md:33-41`。

### 3) 评分与分层协议
- 建议项采用 1-10 关键度。
- 评分锚点：9-10 极关键，7-8 重要业务逻辑，5-6 边界完善，3-4 可选完善，1-2 轻微优化。

证据：`plugins/pr-review-toolkit/agents/pr-test-analyzer.md:42-48`。

### 4) 输出协议
固定栏目：
1. Summary
2. Critical Gaps（8-10）
3. Important Improvements（5-7）
4. Test Quality Issues
5. Positive Observations

证据：`plugins/pr-review-toolkit/agents/pr-test-analyzer.md:49-58`。

### 5) 命令/调用关系
- `review-pr` 根据变更文件决定是否激活该 agent（`plugins/pr-review-toolkit/commands/review-pr.md:35-43`）。
- 命令层可串行或并行执行多个审查角色（`plugins/pr-review-toolkit/commands/review-pr.md:47-56,109-113`）。

### 6) 配置、测试、脚本、文档上下文
- 配置：agent frontmatter；插件 manifest 提供注册元信息（`plugins/pr-review-toolkit/.claude-plugin/plugin.json:1-9`）。
- 测试：本插件未提供“测试覆盖审查 prompt”的回归集。
- 脚本：无测试执行脚本绑定；该 agent 主要基于 diff 与测试文件语义进行审查。
- 文档：README 的触发词、工作流和最佳实践直接对应其职责（`plugins/pr-review-toolkit/README.md:32-51,147-149,201,229,292`）。

## 关键代码路径与文件引用

- Agent 定义：`plugins/pr-review-toolkit/agents/pr-test-analyzer.md:1-69`
- 命令映射：`plugins/pr-review-toolkit/commands/review-pr.md:22-24,35-43,122-126,161-170`
- 用户文档：`plugins/pr-review-toolkit/README.md:32-51,147-149,201,229,292`
- 插件元数据：`plugins/pr-review-toolkit/.claude-plugin/plugin.json:1-9`
- 仓库插件索引：`plugins/README.md:25`
- 市场注册：`.claude-plugin/marketplace.json:117-125`

## 依赖与外部交互

1. 内部依赖
- 依赖可见的 PR 改动与对应测试资产。
- 依赖项目测试约定（提示词引用 `CLAUDE.md`，`plugins/pr-review-toolkit/agents/pr-test-analyzer.md:62`）。

2. 外部交互
- 在命令驱动场景下，间接依赖 `git diff --name-only` 与 `gh pr view`（`plugins/pr-review-toolkit/commands/review-pr.md:31-33`）。
- 不直接调用测试运行器；交互主要是静态审阅与建议输出。

3. 运行前提
- 需要能区分“新增行为”与“现有集成测试覆盖范围”。
- 对无测试仓库或低测试密度仓库，输出将集中在补测建议。

## 风险、边界与改进建议

1. 风险
- 不执行测试导致动态缺陷不可见：该角色无法替代真实 test run。
- 评分异构：1-10 分制与其它 agent（0-100、标签）汇总时需要映射策略。
- 依赖调用方路由准确性：若变更识别漏掉测试相关文件，agent 可能未被触发。

2. 边界
- 只分析覆盖质量与结构风险，不做性能基准测试或端到端环境验证。
- 只给“应测什么、为何要测”的建议，具体用例实现由开发者完成。

3. 改进建议
- 在 `review-pr` 聚合层显式定义 1-10 到 Critical/Important/Suggestion 的映射。
- 给该 agent 增加“可选执行测试摘要输入”（例如最近失败用例），提高建议精度。
- 建立示例回归集，验证对常见漏测模式的稳定识别能力。
