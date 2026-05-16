# FILE `plugins/hookify/hooks/posttooluse.py` 研究文档

## 场景与职责

`posttooluse.py` 是 Hookify 在 `PostToolUse` 事件上的执行器，在工具执行完成后做规则判定与提醒/阻断输出。

它和 `pretooluse.py` 共享相同的规则加载与判定框架，但时机位于“事后阶段”，更适合审计提醒与质量反馈。

## 功能点目的

1. 承接 `PostToolUse` 事件路由。
2. 根据 `tool_name` 复用 `bash/file` 事件过滤逻辑，减少无关规则加载。
3. 将规则命中结果通过 JSON 输出给 Claude runtime，保留上下文反馈。
4. 统一 fail-open：异常不影响主流程继续。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 命令路由与导入

- 路由来源：`hooks.json` 中 `PostToolUse` command（`hooks.json:15-24`）。
- 路径注入：`CLAUDE_PLUGIN_ROOT` + parent dir 入 `sys.path`（`posttooluse.py:13-19`）。
- 导入依赖：`load_rules` 与 `RuleEngine`（`posttooluse.py:22-23`）。

### 2) 执行流程

1. stdin 读取输入 JSON（`posttooluse.py:34`）。
2. 根据 `tool_name` 映射 `event`（`posttooluse.py:37-42`）。
3. `load_rules(event=event)`（`posttooluse.py:45`）。
4. `RuleEngine().evaluate_rules(...)`（`posttooluse.py:48-49`）。
5. stdout 输出 JSON（`posttooluse.py:52`）。
6. 最终 `exit 0`（`posttooluse.py:60-62`）。

### 3) 协议行为

输出协议仍由 `rule_engine` 决定：

- 若 block 且 `hook_event_name=PostToolUse`，返回 `permissionDecision: deny`（`rule_engine.py:72-79`）。
- 若 warn，返回 `systemMessage`（`rule_engine.py:86-91`）。
- 若无命中，返回 `{}`。

该行为与 `PreToolUse` 共享代码路径，但 Post 发生在工具执行后，`deny` 语义主要体现为“事后反馈”，不等同执行前拦截。

### 4) 错误处理

- import 失败：输出 `Hookify import error` 并 `exit 0`（`posttooluse.py:24-27`）。
- 运行异常：输出 `Hookify error` 并 `exit 0`（`posttooluse.py:54-62`）。

## 关键代码路径与文件引用

- 目标文件：`plugins/hookify/hooks/posttooluse.py:1-66`
- 路由配置：`plugins/hookify/hooks/hooks.json:15-24`
- 共享事件映射逻辑对照：`plugins/hookify/hooks/pretooluse.py:45-49`
- 规则加载：`plugins/hookify/core/config_loader.py:198-241`
- 决策输出：`plugins/hookify/core/rule_engine.py:60-94`
- 规则格式来源：`plugins/hookify/README.md:71-242`

## 依赖与外部交互

1. 运行依赖：`python3`、`os/sys/json`。
2. 内部依赖：`hookify.core.config_loader`、`hookify.core.rule_engine`。
3. 环境变量：`CLAUDE_PLUGIN_ROOT`。
4. 输入通道：stdin JSON（事件、工具名、工具输入、hook_event_name）。
5. 输出通道：stdout JSON；stderr 由 `core` 组件打印警告。

## 风险、边界与改进建议

1. Post 阶段 deny 语义边界。
- 风险：工具已执行完毕，`permissionDecision: deny` 可能无法回滚，易与“阻断”预期冲突。
- 建议：为 Post 场景引入单独输出语义（如 `postAction: warn`），避免误导。

2. 与 pre 脚本重复实现。
- 风险：后续修复易出现 pre/post 行为漂移。
- 建议：抽象共享 runner，参数化事件名与映射策略。

3. fail-open 可观测性不足。
- 风险：异常情况下规则失效但仅输出普通 message，监控难。
- 建议：增加可选日志落盘或统计钩子健康状态。

4. 非 Bash/File 工具过滤边界。
- 风险：`event=None` 时只有 `all` 规则可生效，可能遗漏用户预期场景。
- 建议：在 README 明确“未映射工具需使用 `event: all` 或扩展映射表”。
