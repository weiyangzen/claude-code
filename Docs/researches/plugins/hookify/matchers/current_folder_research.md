# DIR `plugins/hookify/matchers` 研究文档

## 场景与职责

`plugins/hookify/matchers` 在当前代码库中是一个“预留的匹配器包边界”，目录仅包含空文件 `__init__.py`，尚未承载任何可执行匹配逻辑。

- 目录现状：`plugins/hookify/matchers/__init__.py` 大小为 0 字节，文件内容为空。
- 调用现状：仓库内未检索到 `from hookify.matchers ...` 或 `import hookify.matchers ...` 的调用。
- 实际匹配职责当前由 `plugins/hookify/core/rule_engine.py` 内部方法承担（工具匹配、字段提取、条件运算、正则匹配）。

因此，这个目录在“运行时职责”上接近空实现，但在“架构职责”上承担了潜在解耦点：它是未来把 `rule_engine` 中匹配策略拆分为独立 matcher 模块的自然落点。

## 功能点目的

结合 `hookify` 插件整体链路，`matchers` 目录当前主要体现为以下目的：

1. 包结构完整性与未来扩展位
- `hookify` 目录按 `commands/agents/skills/hooks/core/matchers/utils` 分层组织。
- `matchers` 作为独立子目录存在，表明项目设计上预留了“匹配策略层”。

2. 与规则模型的语义对齐（但尚未落地代码）
- 规则模型 `Rule` 已包含 `tool_matcher` 字段（`core/config_loader.py`）。
- 引擎中已经有 `_matches_tool` 逻辑（`core/rule_engine.py`）。
- 这意味着“matcher 概念”已经进入数据模型和运行逻辑，但实现还未从 `rule_engine` 抽离到 `matchers` 目录。

3. 对上游命令与文档的语义承接
- 命令层和技能层大量指导用户写 `pattern/conditions`，本质是声明式 matcher 规则。
- 当前声明被 `rule_engine` 直接执行；未来若引入 `matchers` 模块，可在不改变用户规则格式的前提下替换内部匹配实现。

## 具体技术实现（关键流程/数据结构/协议/命令）

虽然 `plugins/hookify/matchers` 目录本身没有实现代码，但其上下文中的匹配实现链路是明确的。

### A. 关键流程（当前实际链路）

1. Claude Code 根据插件 hook 配置触发命令脚本：
- `python3 ${CLAUDE_PLUGIN_ROOT}/hooks/pretooluse.py`
- `python3 ${CLAUDE_PLUGIN_ROOT}/hooks/posttooluse.py`
- `python3 ${CLAUDE_PLUGIN_ROOT}/hooks/stop.py`
- `python3 ${CLAUDE_PLUGIN_ROOT}/hooks/userpromptsubmit.py`

2. hook 执行器读取 stdin JSON，映射事件并调用：
- `load_rules(event=...)`
- `RuleEngine.evaluate_rules(rules, input_data)`

3. `RuleEngine` 内部完成全部“matcher”行为：
- `_matches_tool(matcher, tool_name)`：处理 `*` 与 `|` 分隔匹配。
- `_check_condition(...)`：执行 `regex_match/contains/equals/not_contains/starts_with/ends_with`。
- `_extract_field(...)`：按事件和工具类型抽取 `command/file_path/new_text/transcript/user_prompt` 等字段。
- `_regex_match(...)`：正则匹配（忽略大小写）并使用 LRU cache。

4. 输出 Hook 协议 JSON：
- 工具阶段阻断：`hookSpecificOutput.permissionDecision = deny`
- Stop 阶段阻断：`decision = block`
- 仅提醒：`systemMessage`

结论：目录 `matchers/` 当前不在执行链路上；“匹配器功能”由 `core/rule_engine.py` 内联实现。

### B. 关键数据结构（与 matcher 直接相关）

1. `Condition`（`core/config_loader.py`）
- 字段：`field/operator/pattern`
- 含义：单个匹配条件。

2. `Rule`（`core/config_loader.py`）
- 关键 matcher 相关字段：
  - `conditions: List[Condition]`
  - `pattern`（legacy 简化写法，会转换成 `conditions`）
  - `tool_matcher`（工具名匹配覆盖条件）

3. `RuleEngine`（`core/rule_engine.py`）
- 匹配语义：
  - 规则内 `conditions` 为 AND 关系。
  - `tool_matcher` 先筛选工具，再跑条件。
  - block 优先于 warn。

### C. 协议与命令（matcher 相关约束）

1. Hook 命令入口依赖 `hooks/hooks.json` 声明。
2. 规则来源是项目工作目录下 `.claude/hookify.*.local.md`。
3. `plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh` 把 `matcher` 视为 hook 配置必填字段；但 `hookify/hooks/hooks.json` 当前未为每个事件条目声明该字段，这与通用校验脚本预期存在偏差。

该偏差解释了为什么 `matchers` 目录虽然预留，但现阶段并未承接 hook.json 层级 matcher：项目当前选择了“在 Python 引擎里做二次事件与工具过滤”。

## 关键代码路径与文件引用

### 目录内（目标目录）

- `plugins/hookify/matchers/__init__.py`（空文件，占位包）

### 直接上下文（调用方/被调用方）

- `plugins/hookify/hooks/hooks.json`（事件 -> 命令脚本绑定）
- `plugins/hookify/hooks/pretooluse.py`（PreToolUse 输入适配 + 调用引擎）
- `plugins/hookify/hooks/posttooluse.py`（PostToolUse 输入适配 + 调用引擎）
- `plugins/hookify/hooks/stop.py`（Stop 调用引擎）
- `plugins/hookify/hooks/userpromptsubmit.py`（UserPromptSubmit 调用引擎）
- `plugins/hookify/core/config_loader.py`（`Rule/Condition`、`tool_matcher`、规则加载）
- `plugins/hookify/core/rule_engine.py`（当前 matcher 实现主体）

### 配置/文档/脚本/示例

- `plugins/hookify/README.md`（规则格式、event、operator、即时生效说明）
- `plugins/hookify/commands/hookify.md`（规则创建流程）
- `plugins/hookify/commands/list.md`（规则发现模式）
- `plugins/hookify/commands/configure.md`（规则启停）
- `plugins/hookify/commands/help.md`（用户侧解释）
- `plugins/hookify/skills/writing-rules/SKILL.md`（规则编写规范）
- `plugins/hookify/examples/*.local.md`（规则样例）
- `plugins/plugin-dev/skills/hook-development/SKILL.md`（matcher 语义参考）
- `plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh`（matcher 校验脚本）
- `plugins/plugin-dev/skills/hook-development/scripts/README.md`（hook 测试/校验脚本说明）

## 依赖与外部交互

## 1) 运行时依赖

- Python 3（hooks 中所有命令使用 `python3`）。
- 仅标准库（`re/glob/json/dataclasses/functools` 等），无第三方 matcher 库。
- 环境变量：`CLAUDE_PLUGIN_ROOT`（hook 脚本构建 import path）。

`matchers` 目录自身无直接依赖，因为没有实现代码。

## 2) 外部交互边界

- 输入：Claude Hook runtime 通过 stdin 传 JSON 事件数据。
- 输出：Hook 脚本通过 stdout 返回 JSON 决策结果。
- 文件系统：读取 `.claude/hookify.*.local.md` 和可选 `transcript_path`。

`matchers` 目录当前不直接参与上述 I/O；实际交互由 `hooks/` + `core/` 完成。

## 3) 测试与脚本交互

- `plugins/hookify` 目录下无专属 `tests/` 或 `test/` 自动化测试目录。
- 可复用 `plugin-dev` 的通用脚本进行 schema/行为验证：
  - `validate-hook-schema.sh`
  - `test-hook.sh`
  - `hook-linter.sh`

由于 `matchers` 尚未实现，当前也没有 matcher 级别单测。

## 风险、边界与改进建议

1. 结构预留与实现落点分离，增加维护理解成本
- 现状：`matchers/` 是空目录，但匹配实现全部在 `core/rule_engine.py`。
- 风险：新维护者容易误判目录用途，导致重复实现或错误修改位置。
- 建议：在 `plugins/hookify/matchers/__init__.py` 增加简短模块说明（当前状态 + 未来迁移计划）。

2. matcher 语义分散在多处文档/实现，缺少单一规范源
- 现状：`tool_matcher`、`pattern/conditions`、hook.json 的 matcher 概念分布在不同文件。
- 风险：规则作者、插件维护者、脚本校验器对 matcher 的理解不一致。
- 建议：新增 `plugins/hookify/matchers/README.md`，集中定义：
  - 工具匹配（`tool_matcher`）
  - 条件匹配（`Condition` operators）
  - 与 hook.json `matcher` 的关系与优先级

3. 与通用校验脚本的 schema 预期有偏差
- 现状：`validate-hook-schema.sh` 强制每个 hook 条目有 `matcher`；`hookify/hooks/hooks.json` 当前没有。
- 风险：按通用脚本校验时会报错，影响 CI/本地验证一致性。
- 建议：
  - 要么在 `hookify/hooks/hooks.json` 补充显式 `matcher`（如 `*` 或工具白名单）；
  - 要么调整校验脚本以兼容当前插件 schema。

4. matcher 能力可测试性不足
- 现状：无 matcher 拆分模块，无 matcher 单测。
- 风险：后续演进（如新增 operator、正则策略、字段映射）易出现回归。
- 建议：
  - 第一阶段：先在 `core/rule_engine.py` 增加最小回归测试矩阵；
  - 第二阶段：把 `_matches_tool/_check_condition/_extract_field` 迁移到 `matchers/`，并为每个 matcher 类型建立独立测试。

5. 规则语义边界仍有历史兼容负担
- 现状：simple `pattern`、advanced `conditions`、`tool_matcher` 三套入口并存。
- 风险：配置可读性与调试复杂度上升，且停止/提示词场景字段映射容易误用。
- 建议：在 matcher 层引入显式“事件默认字段映射表”，并在加载阶段输出可读告警，减少静默不生效规则。
