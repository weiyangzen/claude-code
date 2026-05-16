# plugins/pr-review-toolkit/agents/silent-failure-hunter.md 研究

## 场景与职责

`silent-failure-hunter` 是错误处理专项审查器，核心目标是阻止“异常被吞掉但系统继续运行”的隐蔽故障模式。

流程定位：
- 在 `review-pr` 中对应 `errors` 审查面（`plugins/pr-review-toolkit/commands/review-pr.md:24,41,127-131`）。
- README 将其用于错误处理改动、try/catch 复核、PR 收尾前检查（`plugins/pr-review-toolkit/README.md:53-72,226,291`）。

职责边界：
- 重点是识别并解释错误处理缺陷及用户影响，不直接定义产品级错误恢复策略。

## 功能点目的

1. 把“静默失败”上升为强约束
- 首条原则即“silent failures are unacceptable”（`plugins/pr-review-toolkit/agents/silent-failure-hunter.md:14`）。

2. 防止宽捕获与不透明 fallback 掩盖真实错误
- 明确要求审查 catch specificity、fallback justification、error propagation（`plugins/pr-review-toolkit/agents/silent-failure-hunter.md:50-67`）。

3. 强调可观测性与可调试性
- 要求错误日志包含上下文、严重级与可追踪 ID（`plugins/pr-review-toolkit/agents/silent-failure-hunter.md:38-43,90-95`）。

4. 将技术缺陷与用户可感知影响关联
- 输出字段包含 `User Impact` 与 `Hidden Errors`，迫使建议从“代码味道”转为“真实风险”（`plugins/pr-review-toolkit/agents/silent-failure-hunter.md:101-109`）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) Frontmatter 与配置
- `name: silent-failure-hunter`（`plugins/pr-review-toolkit/agents/silent-failure-hunter.md:2`）
- `model: inherit`（`plugins/pr-review-toolkit/agents/silent-failure-hunter.md:4`）
- `color: yellow`（`plugins/pr-review-toolkit/agents/silent-failure-hunter.md:5`）

### 2) 审查流程协议
协议定义 5 段审查流程：
1. 识别所有错误处理代码
2. 对每个处理点做 logging/feedback/specificity/fallback/propagation 复核
3. 单独审查所有用户可见错误消息
4. 搜索隐藏失败模式（空 catch、默认值吞错等）
5. 对齐项目错误处理标准

证据：`plugins/pr-review-toolkit/agents/silent-failure-hunter.md:20-98`。

### 3) 输出协议
每个问题必须包含 7 项：
1. Location
2. Severity（CRITICAL/HIGH/MEDIUM）
3. Issue Description
4. Hidden Errors
5. User Impact
6. Recommendation
7. Example

证据：`plugins/pr-review-toolkit/agents/silent-failure-hunter.md:99-110`。

### 4) 隐含数据结构
- `ErrorHandlingSite`：try/catch、error callback、fallback branch 等位置集合。
- `FailureRisk`：`{severity, hidden_error_types[], user_impact, observability_gap}`。
- `Remediation`：`{code_change, logging_change, message_change, propagation_change}`。

### 5) 特殊项目约束
Special Considerations 写死了项目假设：
- 日志函数：`logForDebugging`, `logError`, `logEvent`
- Sentry 错误 ID 路径：`constants/errorIds.ts`

证据：`plugins/pr-review-toolkit/agents/silent-failure-hunter.md:123-126`。

### 6) 配置、测试、脚本、文档上下文
- 配置：frontmatter + 插件 manifest（`plugins/pr-review-toolkit/.claude-plugin/plugin.json:1-9`）。
- 测试：无“错误处理规则契约测试”资产。
- 脚本：无专属脚本，依赖命令层与运行时读取代码上下文。
- 文档：README 对触发词、使用时机与工作流有明确定义（`plugins/pr-review-toolkit/README.md:53-72,150-152,203,226,291`）。

## 关键代码路径与文件引用

- Agent 定义：`plugins/pr-review-toolkit/agents/silent-failure-hunter.md:1-130`
- 命令映射：`plugins/pr-review-toolkit/commands/review-pr.md:24,41,127-131,161-170`
- 用户文档：`plugins/pr-review-toolkit/README.md:53-72,150-152,203,226,291`
- 插件元数据：`plugins/pr-review-toolkit/.claude-plugin/plugin.json:1-9`
- 插件索引：`plugins/README.md:25`
- 市场注册：`.claude-plugin/marketplace.json:117-125`

## 依赖与外部交互

1. 内部依赖
- 依赖上游准确识别“错误处理改动”并触发该 agent。
- 依赖项目的日志与监控约定作为判定基线。

2. 外部交互
- 间接依赖 `git diff --name-only`、`gh pr view` 的变更与 PR 上下文探测（由 `review-pr` 执行）。
- 与最终用户体验强相关：输出直接影响错误提示策略与监控埋点。

3. 运行前提
- 代码中存在可解析的错误处理路径（异常、返回码、回调）。
- 需要有可参考的日志/错误 ID 规范，否则建议会退化为通用风格。

## 风险、边界与改进建议

1. 风险
- 项目特定假设过强：日志函数名与 `constants/errorIds.ts` 并非通用约定，跨项目易误报。
- 高敏感策略可能放大噪声：对“记录后继续”场景若无业务上下文，可能过度判定。
- 只做静态审阅：无法捕获运行时环境触发的隐式错误链。

2. 边界
- 该 agent 不负责定义完整容灾架构，只关注当前改动中的静默失败与可观测性缺陷。
- 输出示例代码为建议模板，不等同可直接应用的生产修复。

3. 改进建议
- 将日志函数与 errorIds 路径改为可配置变量，默认给出占位而非硬编码。
- 在输出中加入“误报风险说明”字段（例如需业务确认的 fallback 场景）。
- 结合 `pr-test-analyzer` 增加“错误路径是否已有回归测试”联动检查。
