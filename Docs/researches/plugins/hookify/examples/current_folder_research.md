# DIR `plugins/hookify/examples` 研究文档

## 场景与职责

`plugins/hookify/examples` 是 Hookify 的“规则模板样例库”，职责不是直接参与运行时匹配，而是给用户、命令文档和技能文档提供可复制的 `.local.md` 规则参考。

该目录当前包含 4 个示例：
- `dangerous-rm.local.md`：演示 `bash` 事件 + `block` 行为。
- `console-log-warning.local.md`：演示 `file` 事件 + `warn` 行为。
- `sensitive-files-warning.local.md`：演示 `conditions` 多字段结构（当前示例只有 1 条条件）。
- `require-tests-stop.local.md`：演示 `stop` 事件与“结束前检查”场景，默认禁用。

上下文定位：
- 文档侧将其作为“可参考模板”：`plugins/hookify/commands/help.md:175`、`plugins/hookify/commands/list.md:81`、`plugins/hookify/skills/writing-rules/SKILL.md:324`。
- 运行侧实际加载的是项目根目录 `.claude/hookify.*.local.md`：`plugins/hookify/core/config_loader.py:210`。

## 功能点目的

### 1) `dangerous-rm.local.md`
目的：在 `Bash` 场景下识别高风险删除命令并阻断执行。
- 关键字段：`event: bash`、`pattern: rm\s+-rf`、`action: block`（`plugins/hookify/examples/dangerous-rm.local.md:4-6`）。
- 适配场景：防止误删、越权批量删除。

### 2) `console-log-warning.local.md`
目的：在文件修改阶段提醒调试代码残留风险，不中断流程。
- 关键字段：`event: file`、`pattern: console\.log\(`、`action: warn`（`plugins/hookify/examples/console-log-warning.local.md:4-6`）。
- 适配场景：代码整洁性与发布质量提醒。

### 3) `sensitive-files-warning.local.md`
目的：基于文件路径特征识别敏感文件操作并告警。
- 关键字段：`event: file` + `conditions` + `regex_match`（`plugins/hookify/examples/sensitive-files-warning.local.md:4-9`）。
- 适配场景：`.env`、凭证、secret 相关文件编辑风险提示。

### 4) `require-tests-stop.local.md`
目的：在会话 `Stop` 事件前做“是否跑测”的流程约束（默认关闭）。
- 关键字段：`enabled: false`、`event: stop`、`action: block`、`field: transcript`（`plugins/hookify/examples/require-tests-stop.local.md:3-9`）。
- 适配场景：任务完成前质量闸门。

## 具体技术实现（关键流程/数据结构/协议/命令）

### A. 关键流程（从示例到生效）
1. 用户通过 `/hookify` 或手工方式把规则放到项目 `.claude/` 目录，命名为 `hookify.{name}.local.md`（`plugins/hookify/commands/hookify.md:84-137`）。
2. Hook 入口由 `hooks.json` 注册，Claude 在 `PreToolUse/PostToolUse/Stop/UserPromptSubmit` 调用 Python 执行器（`plugins/hookify/hooks/hooks.json:4-47`）。
3. 执行器从 stdin 读取 JSON 输入，按工具类型映射事件（如 Bash->`bash`，Edit/Write/MultiEdit->`file`），调用 `load_rules(event=...)`（`plugins/hookify/hooks/pretooluse.py:39-53`，`posttooluse.py:34-46`，`stop.py:34-38`，`userpromptsubmit.py:34-38`）。
4. `load_rules` 扫描 `.claude/hookify.*.local.md` 并解析 frontmatter（`plugins/hookify/core/config_loader.py:198-241`）。
5. `RuleEngine.evaluate_rules` 对规则求值，输出 Hook 协议 JSON（`plugins/hookify/core/rule_engine.py:35-94`）。

结论：`plugins/hookify/examples/*.local.md` 本身只是模板，不会被 `load_rules` 直接扫描。

### B. 数据结构
- `Condition`：`field/operator/pattern`（`plugins/hookify/core/config_loader.py:15-30`）。
- `Rule`：`name/enabled/event/pattern/conditions/action/tool_matcher/message`（`plugins/hookify/core/config_loader.py:32-84`）。
- 简单 `pattern` 会按 `event` 自动降解成单条件：
  - `bash -> field=command`
  - `file -> field=new_text`
  - 其他 -> `field=content`
  （`plugins/hookify/core/config_loader.py:56-73`）

### C. 匹配与协议
- 条件匹配算子：`regex_match/contains/equals/not_contains/starts_with/ends_with`（`plugins/hookify/core/rule_engine.py:166-180`）。
- `regex_match` 使用 LRU 缓存编译（`maxsize=128`）与 `IGNORECASE`（`plugins/hookify/core/rule_engine.py:13-25`）。
- 阻断输出协议：
  - `Stop`：`{"decision":"block","reason":...,"systemMessage":...}`（`plugins/hookify/core/rule_engine.py:66-71`）。
  - `PreToolUse/PostToolUse`：`hookSpecificOutput.permissionDecision = deny`（`plugins/hookify/core/rule_engine.py:72-79`）。
  - 其他事件（含 `UserPromptSubmit`）仅回 `systemMessage`（`plugins/hookify/core/rule_engine.py:80-84`）。
- 告警输出协议：仅 `systemMessage`（`plugins/hookify/core/rule_engine.py:86-91`）。

### D. 关键命令与落地方式
- Hook 执行命令（注册态）：
  - `python3 ${CLAUDE_PLUGIN_ROOT}/hooks/pretooluse.py`
  - `python3 ${CLAUDE_PLUGIN_ROOT}/hooks/posttooluse.py`
  - `python3 ${CLAUDE_PLUGIN_ROOT}/hooks/stop.py`
  - `python3 ${CLAUDE_PLUGIN_ROOT}/hooks/userpromptsubmit.py`
  （见 `plugins/hookify/hooks/hooks.json:9,20,31,42`）
- 示例复制到项目生效目录（手工）：
  - `mkdir -p .claude`
  - `cp ${CLAUDE_PLUGIN_ROOT}/examples/dangerous-rm.local.md .claude/hookify.dangerous-rm.local.md`

## 关键代码路径与文件引用

### 目标目录（样例本体）
- `plugins/hookify/examples/dangerous-rm.local.md`
- `plugins/hookify/examples/console-log-warning.local.md`
- `plugins/hookify/examples/sensitive-files-warning.local.md`
- `plugins/hookify/examples/require-tests-stop.local.md`

### 直接调用方/消费方
- 规则创建入口：`plugins/hookify/commands/hookify.md`
- 帮助与引导引用 examples：
  - `plugins/hookify/commands/help.md:175`
  - `plugins/hookify/commands/list.md:81`
  - `plugins/hookify/skills/writing-rules/SKILL.md:324-327`
- 使用说明与示例片段：`plugins/hookify/README.md:71-208`

### 运行时被调用链
- Hook 注册：`plugins/hookify/hooks/hooks.json`
- 执行器：
  - `plugins/hookify/hooks/pretooluse.py`
  - `plugins/hookify/hooks/posttooluse.py`
  - `plugins/hookify/hooks/stop.py`
  - `plugins/hookify/hooks/userpromptsubmit.py`
- 规则解析：`plugins/hookify/core/config_loader.py`
- 规则求值：`plugins/hookify/core/rule_engine.py`

## 依赖与外部交互

### 运行依赖
- Python 3（README 声明 `3.7+`）：`plugins/hookify/README.md:300`。
- 仅 Python 标准库：`os/sys/glob/re/json/dataclasses/functools/typing`（见 `core` 与 `hooks` 脚本 import）。

### 外部输入输出
- 输入：Claude Hook Runtime 通过 stdin 提供 JSON（`json.load(sys.stdin)`）。
- 输出：执行器向 stdout 输出 JSON 协议并始终 `exit(0)`，避免因 hook 异常硬阻断（`pretooluse.py:58-70` 等）。
- 环境变量：`CLAUDE_PLUGIN_ROOT` 用于注入 `sys.path`（`plugins/hookify/hooks/pretooluse.py:14-23` 等）。
- 文件系统交互：
  - 读取规则：`.claude/hookify.*.local.md`（`config_loader.py:210-211`）。
  - `stop` 场景可读取 `transcript_path` 文件内容（`rule_engine.py:207-225`）。

### 测试/脚本现状
- `plugins/hookify` 目录无独立自动化测试（无 `tests/`、无 pytest 入口）。
- 仅在 `config_loader.py`、`rule_engine.py` 内提供 `__main__` 手工测试片段（`config_loader.py:277-297`，`rule_engine.py:276-313`）。

## 风险、边界与改进建议

### 风险 1：示例文件“看似可用”但默认不会自动生效
- 现状：示例位于插件目录且文件名无 `hookify.` 前缀，不满足运行时扫描模式。
- 影响：用户直接修改 `plugins/hookify/examples/*` 可能误以为规则已启用。
- 建议：
  - 在 examples 顶部增加醒目注释“需复制到 `.claude/hookify.*.local.md` 才生效”；
  - 增加一键安装脚本（如 `scripts/install-example-rule.sh`）。

### 风险 2：`require-tests-stop` 的 `not_contains` 与“多候选命令”写法语义不符
- 现状：`pattern: npm test|pytest|cargo test` 被当普通字符串做子串判断（`rule_engine.py:172-173`），不是正则 OR。
- 影响：规则启用后可能长期误判为“未跑测试”并阻断 stop。
- 建议：
  - 改成 `operator: regex_match` + 负向逻辑支持（如新增 `not_regex_match`）；
  - 或把 transcript 判定改为多条件 + OR 语义（当前仅 AND）。

### 风险 3：`UserPromptSubmit` 事件下 `block` 行为协议不明确
- 现状：`evaluate_rules` 对 `UserPromptSubmit` 进入“其他事件”分支，仅返回 `systemMessage`（`rule_engine.py:80-84`）。
- 影响：`event: prompt + action: block` 可能只提示不阻断。
- 建议：补充 `UserPromptSubmit` 的显式阻断协议映射与回归测试。

### 风险 4：frontmatter 解析器为自实现 YAML 子集
- 现状：`extract_frontmatter` 手写解析（`config_loader.py:87-195`）。
- 影响：复杂 YAML（多层嵌套、转义、数组字面量）兼容性有限。
- 建议：
  - 增加解析失败用例测试；
  - 若允许依赖，可替换为标准 YAML 解析器；
  - 至少在 docs 中明确“支持子集”。

### 风险 5：缺少 examples -> runtime 的端到端回归测试
- 现状：无自动验证示例文件是否与 `RuleEngine` 行为一致。
- 建议：新增最小测试矩阵：
  - `bash/file/stop/prompt` 各一条；
  - 覆盖 `warn/block`、`pattern/conditions`、协议字段断言；
  - 将 `examples/*.local.md` 作为 fixture 复制到临时 `.claude/` 后执行。

