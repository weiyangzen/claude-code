# FILE `plugins/agent-sdk-dev/agents/agent-sdk-verifier-py.md` 研究文档

## 场景与职责

`agent-sdk-verifier-py.md` 是 `agent-sdk-dev` 插件中的 Python 验收代理协议文件，用于在 Agent SDK 项目创建或修改后执行“可部署前检查”。其 frontmatter 将该能力注册为可调用 agent：`name: agent-sdk-verifier-py`、`model: sonnet`（`plugins/agent-sdk-dev/agents/agent-sdk-verifier-py.md:1-5`）。

该文件不直接实现 Python 代码，而是通过提示词协议定义审查流程、审查范围和报告格式。调用方来自 `/new-sdk-app` 命令在验证阶段的语言分支（`plugins/agent-sdk-dev/commands/new-sdk-app.md:128-135`），同时 README 对外承诺“Python 项目创建后自动触发 verifier”（`plugins/agent-sdk-dev/README.md:50-73`）。

职责边界：
- 负责 SDK 使用正确性、环境可复现性、安全配置、文档完整性。
- 明确不负责通用代码风格争议（PEP8、命名、导入排序等）（`plugins/agent-sdk-dev/agents/agent-sdk-verifier-py.md:73-78`）。

## 功能点目的

该文件按 8 个 Focus 维度定义验收目的（`plugins/agent-sdk-dev/agents/agent-sdk-verifier-py.md:13-71`）：

1. SDK 安装与配置：确认 `claude-agent-sdk` 已安装、版本不过旧、Python 版本满足通常要求、虚拟环境建议可见（`:15-19`）。
2. Python 环境可复现：检查 `requirements.txt` 或 `pyproject.toml`、依赖声明与版本约束（`:20-26`）。
3. SDK 使用模式：检查 `claude_agent_sdk` 导入、初始化、参数、响应模式、权限、MCP 集成（`:27-35`）。
4. 代码可运行基础：检查导入、语法和最小错误处理（`:37-42`）。
5. 安全与环境：`.env.example`、`.gitignore`、密钥不硬编码、API 调用错误处理（`:44-49`）。
6. 官方最佳实践对齐：系统提示词、模型选择、权限范围、工具与子代理、会话处理（`:51-58`）。
7. 功能流程完整性：初始化与执行链路是否闭合、SDK 特定错误是否覆盖（`:60-65`）。
8. 文档可交接：README、安装步骤、虚拟环境说明、定制配置说明（`:67-71`）。

这些功能点共同目标是把“脚手架可生成”提升为“项目可交付验收”。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 关键流程

该 verifier 的执行协议是四步式（`plugins/agent-sdk-dev/agents/agent-sdk-verifier-py.md:80-104`）：

1. 读取目标文件集：依赖声明、主程序、环境文件、配置文件（`:82-87`）。
2. 对照官方文档：要求使用 WebFetch 读取 Python SDK 官方文档并比对偏差（`:89-93`）。
3. 做导入/语法层验证：确认导入正确、无明显语法问题（`:95-99`）。
4. 分析 SDK 调用：检查调用参数和模式与官方示例一致性（`:101-104`）。

与上游命令的耦合点在于：`/new-sdk-app` 在“生成并安装后”调用该 verifier，因此 verifier 默认工作在“已有项目文件”的上下文，而不是空目录（`plugins/agent-sdk-dev/commands/new-sdk-app.md:81-93,128-135`）。

### 数据结构与协议

1. Agent 注册结构（YAML frontmatter）
- `name`：运行时调用标识。
- `description`：触发场景约束（创建/修改后）。
- `model`：执行模型固定为 `sonnet`。
- 证据：`plugins/agent-sdk-dev/agents/agent-sdk-verifier-py.md:1-5`。

2. 报告结构（输出协议）
- 状态枚举：`PASS | PASS WITH WARNINGS | FAIL`。
- 分段：`Summary`、`Critical Issues`、`Warnings`、`Passed Checks`、`Recommendations`。
- 证据：`plugins/agent-sdk-dev/agents/agent-sdk-verifier-py.md:106-138`。

3. 审查边界协议
- `What NOT to Focus On` 明确排除一般代码风格争论，防止验收结果偏离 SDK 合规目标。
- 证据：`plugins/agent-sdk-dev/agents/agent-sdk-verifier-py.md:73-78`。

### 关键命令与检查动作

该文件本身未强制具体 shell 命令模板，但要求完成“导入与语法”检查（`plugins/agent-sdk-dev/agents/agent-sdk-verifier-py.md:95-100`）。与之配套的上游命令文档要求安装/确认 SDK：
- `pip install claude-agent-sdk`
- `pip show claude-agent-sdk`
- 证据：`plugins/agent-sdk-dev/commands/new-sdk-app.md:84-87`。

因此 Python verifier 当前属于“策略约束 > 命令约束”，相比 TS verifier 的 `npx tsc --noEmit` 可执行性更弱。

## 关键代码路径与文件引用

核心被研究文件：
- `plugins/agent-sdk-dev/agents/agent-sdk-verifier-py.md:1-140`

直接调用方：
- `plugins/agent-sdk-dev/commands/new-sdk-app.md:128-135`（按语言触发 verifier）

能力说明与用户预期：
- `plugins/agent-sdk-dev/README.md:50-81`（Python verifier 检查项与输出）
- `plugins/agent-sdk-dev/README.md:69-73`（自动触发说明）

插件注册与发现：
- `plugins/agent-sdk-dev/.claude-plugin/plugin.json:1-9`
- `.claude-plugin/marketplace.json:12-16`
- `plugins/README.md:15`

同目录相关文件：
- `plugins/agent-sdk-dev/agents/agent-sdk-verifier-ts.md:1-145`（同类协议、可对比）

测试与脚本上下文：
- `plugins/agent-sdk-dev` 当前仅含 5 个文档/元数据文件，无测试代码与自动化脚本（`find plugins/agent-sdk-dev -maxdepth 3 -type f`）。

## 依赖与外部交互

内部依赖：
- 依赖 `/new-sdk-app` 在验证阶段分支调用；若上游不触发，该 agent 不会自动执行（`plugins/agent-sdk-dev/commands/new-sdk-app.md:128-135`）。
- 依赖插件 README 对外行为承诺，影响用户预期一致性（`plugins/agent-sdk-dev/README.md:50-73`）。

外部依赖：
- 官方文档站：`https://docs.claude.com/en/api/agent-sdk/python`（通过 WebFetch）（`plugins/agent-sdk-dev/agents/agent-sdk-verifier-py.md:91`）。
- Python 生态工具链：`pip`、虚拟环境、依赖锁定文件。

与目标工程的文件系统交互（被审查对象）：
- `requirements.txt`/`pyproject.toml`
- `main.py`/`app.py`/`src/*`
- `.env.example`、`.gitignore`
- 证据：`plugins/agent-sdk-dev/agents/agent-sdk-verifier-py.md:82-87`。

## 风险、边界与改进建议

风险：
1. 规则执行一致性风险：当前为自然语言提示词约束，缺少机器强校验框架，结果可能受执行会话波动影响。
2. 可复现性风险：仅要求“检查导入和语法”，未给固定命令模板，跨会话结果可比较性较弱（`plugins/agent-sdk-dev/agents/agent-sdk-verifier-py.md:95-100`）。
3. 网络依赖风险：官方文档对照依赖 WebFetch，离线环境下最佳实践对齐能力下降（`plugins/agent-sdk-dev/agents/agent-sdk-verifier-py.md:91-93`）。
4. 模型策略固化风险：`model: sonnet` 固定写死，未来模型策略切换时维护成本上升（`plugins/agent-sdk-dev/agents/agent-sdk-verifier-py.md:4`）。

边界：
- 不负责项目创建，只负责项目验收；创建逻辑在 `commands/new-sdk-app.md`。
- 不负责 Python 风格规范裁决，只关注 SDK 合规与可运行性。

改进建议：
1. 增加 Python 标准校验命令模板
- 在文档中固化最小命令，例如 `python -m py_compile`、`python -c "import claude_agent_sdk"`，提升复现性。
2. 增加机器可读报告附录
- 在保留文本报告同时输出 JSON 结构（状态、问题数组、证据路径），便于自动化流水线消费。
3. 补充“Critical 判定标准”
- 定义阻断发布的硬条件（例如导入失败、密钥泄露、核心初始化错误）和可延后项，减少判定漂移。
4. 将模型字段改为可配置注入
- 允许通过插件设置或组织策略覆盖默认模型，降低后续升级摩擦。
