# FILE `plugins/hookify/hooks/stop.py` 研究文档

## 场景与职责

`stop.py` 是 Hookify 在 `Stop` 事件上的执行器，在主代理准备结束任务时触发，用于执行“收尾质量门禁”（例如未跑测试不允许结束）。

它是 Hookify 中最接近“完成条件控制”的入口，直接关联会话是否允许结束。

## 功能点目的

1. 处理会话停止前校验。
2. 仅加载 `event='stop'` 规则，避免无关规则开销。
3. 通过规则引擎输出 Stop 协议（`decision: block/approve` 语义中的 block 分支）。
4. 继续采用 fail-open，确保 hook 自身异常不阻断系统。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 路由与导入

- 路由配置：`hooks.json` 的 `Stop` command（`hooks.json:26-35`）。
- 路径注入：`CLAUDE_PLUGIN_ROOT` + parent dir（`stop.py:13-19`）。
- 导入依赖：`load_rules` / `RuleEngine`（`stop.py:22-23`）。

### 2) 执行流程

1. stdin 读取事件输入（`stop.py:34`）。
2. 固定 `rules = load_rules(event='stop')`（`stop.py:37`）。
3. 调用 `RuleEngine.evaluate_rules`（`stop.py:40-41`）。
4. 输出 JSON（`stop.py:44`）。
5. `finally` 始终 `exit 0`（`stop.py:53-55`）。

### 3) Stop 协议实现

当命中 `action: block` 规则且 `hook_event_name=Stop` 时，引擎返回：

- `decision: "block"`
- `reason: <合并消息>`
- `systemMessage: <合并消息>`

实现：`plugins/hookify/core/rule_engine.py:66-71`。

### 4) Stop 特有字段访问

规则可访问 `reason` 与 `transcript` 字段：

- `reason` 从 `input_data.reason` 读取
- `transcript` 从 `input_data.transcript_path` 指向的文件读取全文

实现：`rule_engine.py:203-225`。

这使 Stop 规则可以根据整个会话内容做结束判定。

## 关键代码路径与文件引用

- 目标文件：`plugins/hookify/hooks/stop.py:1-59`
- 路由配置：`plugins/hookify/hooks/hooks.json:26-35`
- Stop 输出协议：`plugins/hookify/core/rule_engine.py:66-71`
- transcript 字段读取：`plugins/hookify/core/rule_engine.py:207-225`
- stop 规则配置样例：
  - `plugins/hookify/README.md:189-208`
  - `plugins/hookify/examples/require-tests-stop.local.md`

## 依赖与外部交互

1. 输入依赖：stdin JSON（含 `hook_event_name`、可能含 `reason/transcript_path`）。
2. 文件系统依赖：读取 `.claude/hookify.*.local.md` 与可能的 transcript 文件。
3. 环境依赖：`CLAUDE_PLUGIN_ROOT`、`python3`。
4. 协议交互：stdout JSON 返回 Stop 决策；stderr 输出读取 transcript 失败警告。

## 风险、边界与改进建议

1. README 示例与运算符语义可能产生偏差。
- 现状：README Stop 示例使用 `operator: not_contains` + `pattern: npm test|pytest|cargo test`（`README.md:197-201`）。
- 风险：`not_contains` 是字面包含，不是正则 OR，可能与用户预期不一致。
- 建议：示例改为 `regex_match`/`not_regex_match`（需新增运算符）或明确语义说明。

2. transcript 大文件性能边界。
- 风险：Stop 阶段规则多次读取 transcript 可能增加延迟。
- 建议：在单次 evaluate 内缓存 transcript 文本，避免重复 I/O。

3. fail-open 策略对“强收尾门禁”不够严格。
- 风险：解析异常时会话仍可结束，规则形同失效。
- 建议：提供 `STRICT_STOP_HOOK=1` 之类开关，允许 stop 场景 fail-closed。

4. 异常观测不足。
- 风险：当前仅返回 `systemMessage`，难做集中排障。
- 建议：增加统一错误码/阶段标签并输出到可收集日志通道。
