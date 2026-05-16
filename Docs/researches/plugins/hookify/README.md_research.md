# FILE `plugins/hookify/README.md` 研究文档

## 场景与职责

`plugins/hookify/README.md` 是 Hookify 插件的用户面主文档，承担“产品说明 + 规则 DSL 文档 + 操作手册 + 故障排查”的复合职责。

在整个插件链路中，它位于最上层：
- 对外定义能力边界：通过 `/hookify` 系命令创建和管理规则（`plugins/hookify/README.md:39-69`）。
- 对内约束配置契约：定义 `.claude/hookify.*.local.md` frontmatter 字段与 message 语义（`plugins/hookify/README.md:71-119,235-257`）。
- 连接运行时实现：把“即时生效、无重启”承诺映射到 Hook 执行器每次动态加载规则这一事实（`plugins/hookify/README.md:28-29,309-310`；`plugins/hookify/core/config_loader.py:198-241`）。

它的调用关系是“被人读、被命令文档复述、被实现验证”：
- 调用方：用户、`/hookify:help` 内容、插件目录总览文档（`plugins/hookify/commands/help.md:10-76`，`plugins/README.md:22`）。
- 被调用方：`commands/*.md`（交互流程）、`hooks/hooks.json`（事件入口）、`core/*`（规则解析与判定）。

## 功能点目的

README 的核心功能点与目的可分为 7 组：

1. 快速上手与价值主张（`plugins/hookify/README.md:16-35`）
- 目的：把“从一句自然语言生成规则并立刻验证”作为主路径，降低上手门槛。

2. 命令路由说明（`plugins/hookify/README.md:37-69`）
- 目的：定义四个命令职责分工。
- 与实现一致性：对应 `commands/hookify.md`、`list.md`、`configure.md`、`help.md`。

3. 规则格式规范（`plugins/hookify/README.md:71-119`）
- 目的：给出 simple pattern 与 advanced conditions 两种写法，覆盖入门和高级场景。
- 关键字段：`name/enabled/event/pattern|conditions/action`。

4. 事件类型/字段/操作符参考（`plugins/hookify/README.md:122-129,235-257`）
- 目的：把规则 DSL 映射到 Hook 输入 JSON 字段，指导用户选择可匹配字段。

5. 示例驱动说明（`plugins/hookify/README.md:150-208`）
- 目的：通过“危险命令拦截、调试代码告警、停止前测试检查”覆盖典型治理问题。

6. 管理与排障（`plugins/hookify/README.md:261-325`）
- 目的：强调规则启停和排障步骤，降低“规则不生效”的排查成本。

7. 安装与依赖声明（`plugins/hookify/README.md:289-301`）
- 目的：声明插件发现方式和运行依赖（Python 3.7+，stdlib only）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) README 定义的产品流程如何落地

README 所述流程与代码实现的对应关系：

1. 用户通过 `/hookify` 或手工创建 `.claude/hookify.*.local.md`（`plugins/hookify/README.md:41-52,75-119`）。
2. Claude Code 在事件触发时调用 `hooks/hooks.json` 注册的 Python 脚本（`plugins/hookify/hooks/hooks.json:4-47`）。
3. Hook 脚本按工具推断 event（bash/file）或固定 event（stop/prompt），然后 `load_rules(event=...)`（`plugins/hookify/hooks/pretooluse.py:42-53`，`stop.py:36-41`，`userpromptsubmit.py:36-41`）。
4. `config_loader` 扫描 `.claude/hookify.*.local.md`，解析 frontmatter 为 `Rule/Condition`（`plugins/hookify/core/config_loader.py:15-84,198-241`）。
5. `RuleEngine` 进行条件 AND 判定并生成 Hook 协议输出（`plugins/hookify/core/rule_engine.py:35-95,120-125`）。

这直接支撑 README 的“规则下一次工具调用即生效、无需重启”主张。

### 2) README DSL 与真实数据结构映射

README 字段 -> Python 数据结构：
- `conditions[].field/operator/pattern` -> `Condition`（`plugins/hookify/core/config_loader.py:15-29`）。
- `name/enabled/event/pattern/action/tool_matcher/message` -> `Rule`（`plugins/hookify/core/config_loader.py:33-43,75-84`）。
- simple `pattern` 会被自动转成单条件：
  - `event=bash` -> `field=command`
  - `event=file` -> `field=new_text`
  - 其他 event -> `field=content`
  （`plugins/hookify/core/config_loader.py:57-73`）

### 3) README 行为语义与协议输出

README 说明 `warn`/`block` 两种动作（`plugins/hookify/README.md:93-96,147-149`），在规则引擎里映射为：
- Stop 阻断：`{"decision":"block","reason":...,"systemMessage":...}`（`plugins/hookify/core/rule_engine.py:66-71`）
- Pre/PostToolUse 阻断：`permissionDecision: deny`（`plugins/hookify/core/rule_engine.py:72-79`）
- warn：只返回 `systemMessage`（`plugins/hookify/core/rule_engine.py:86-91`）

### 4) README 示例与实现细节的关键差异

1. Stop 示例中的 `not_contains` 使用了“正则样式 OR 串”（`npm test|pytest|cargo test`），但实现是纯子串比较（`pattern not in field_value`），不会按正则 OR 解释（`plugins/hookify/README.md:198-201`；`plugins/hookify/core/rule_engine.py:172-173`）。这会导致误判风险。

2. README 将 stop 事件描述为“通用匹配 session state”（`plugins/hookify/README.md:258-260`），但 simple `pattern` 默认映射 `field=content`，而 stop 场景并未提供 `content` 字段提取，需使用 `conditions` + `field: transcript|reason` 才可靠（`plugins/hookify/core/config_loader.py:67`；`plugins/hookify/core/rule_engine.py:202-229`）。

3. README 强调“无重启”，其技术前提是每次事件都实时扫描 `.claude/hookify.*.local.md`；因此规则数量和复杂正则会直接影响 hook 执行时延（`plugins/hookify/core/config_loader.py:209-227`；`plugins/hookify/core/rule_engine.py:13-25,266-273`）。

### 5) README 与命令/Agent/Skill 的契约一致性

- `README` 的命令说明与 `commands/*.md` 基本一致（`plugins/hookify/README.md:55-69` 对应 `plugins/hookify/commands/*.md`）。
- `/hookify` 文档要求优先加载 `writing-rules` skill，确保生成规则符合 README DSL（`plugins/hookify/commands/hookify.md:9`；`plugins/hookify/skills/writing-rules/SKILL.md:13-99`）。
- 无参 `/hookify` 通过 `conversation-analyzer` 抽取问题模式，回填到 README 规定格式（`plugins/hookify/commands/hookify.md:24-58`；`plugins/hookify/agents/conversation-analyzer.md:47-76`）。

## 关键代码路径与文件引用

目标文件：
- `plugins/hookify/README.md`

调用方（谁依赖/复述该文档）：
- `plugins/hookify/commands/help.md:10-159`（帮助命令几乎是 README 的精简复述）
- `plugins/README.md:22`（插件目录级索引入口）

被调用方（README 指向的落地实现）：
- 命令与交互：
  - `plugins/hookify/commands/hookify.md`
  - `plugins/hookify/commands/list.md`
  - `plugins/hookify/commands/configure.md`
  - `plugins/hookify/commands/help.md`
- Hook 注册与执行：
  - `plugins/hookify/hooks/hooks.json`
  - `plugins/hookify/hooks/pretooluse.py`
  - `plugins/hookify/hooks/posttooluse.py`
  - `plugins/hookify/hooks/stop.py`
  - `plugins/hookify/hooks/userpromptsubmit.py`
- 规则解析与匹配：
  - `plugins/hookify/core/config_loader.py`
  - `plugins/hookify/core/rule_engine.py`
- 支撑资料：
  - `plugins/hookify/skills/writing-rules/SKILL.md`
  - `plugins/hookify/agents/conversation-analyzer.md`
  - `plugins/hookify/examples/*.local.md`

配置/测试/脚本/文档依赖（按要求覆盖）：
- 配置：`hooks/hooks.json`、`.claude/hookify.*.local.md`。
- 测试：插件内无独立测试目录，主要靠示例文件与手工触发验证。
- 脚本：Hook 脚本是 README“即时生效”承诺的执行主体；仓库 `.ops/*` 不参与插件运行。
- 文档：`commands/help.md` 与 `skills/writing-rules/SKILL.md` 是 README 的操作化文档。

## 依赖与外部交互

1. 运行时依赖
- Python 3.7+（README 声明，`plugins/hookify/README.md:300`）。
- 标准库为主（`json/os/sys/glob/re/dataclasses/functools` 等）。

2. 外部交互面
- 与 Claude Code Hook 协议交互：stdin 接收事件 JSON，stdout 返回决策 JSON（`plugins/hookify/hooks/*.py`）。
- 与文件系统交互：读取项目 `.claude/hookify.*.local.md` 及 stop 场景 transcript 文件（`plugins/hookify/core/config_loader.py:209-211`；`plugins/hookify/core/rule_engine.py:207-225`）。

3. 环境变量依赖
- `CLAUDE_PLUGIN_ROOT` 用于 hook 脚本构建 `sys.path`，保障 `hookify.core` 可导入（`plugins/hookify/hooks/pretooluse.py:14-23` 等）。

4. 网络/第三方依赖
- 无网络调用、无第三方 Python 包（与 README 声明一致，`plugins/hookify/README.md:301`）。

## 风险、边界与改进建议

1. 风险：README 与实现在部分语义上有偏差（高优先）。
- `not_contains` 示例写法易被误解为正则 OR；实际是字面子串判断。
- 建议：README 增加“`not_contains` 非正则”警告，或新增 `not_regex_match` 操作符。

2. 风险：stop/prompt 场景的 simple `pattern` 可用性不足。
- 现状：默认映射到 `content`，但运行时通常不提供该字段。
- 建议：
  - 文档层强制推荐 stop/prompt 使用 `conditions`；
  - 代码层把 stop 默认映射到 `transcript`、prompt 默认映射到 `user_prompt`。

3. 风险：性能说明不够量化。
- README 只给了“复杂 regex 会慢”提示（`plugins/hookify/README.md:321-324`）。
- 建议：补充建议上限（如规则数/单规则复杂度）和诊断方式（例如记录匹配耗时）。

4. 风险：规则目录边界被误解。
- README 虽提到规则在项目 `.claude/`，但用户仍可能写到插件目录。
- 建议：在 Quick Start 中加入“当前工作目录示例路径”，并提醒在项目 `.gitignore` 排除 `.claude/*.local.*`。

5. 边界：README 是用户契约，不是执行保证。
- 真正行为由 `hooks/*.py` 与 `core/*` 决定；若代码升级未同步文档会产生认知漂移。
- 建议：在 CI 增加文档-实现一致性检查（字段、operator、event 映射）。

6. 改进：补充最小验证流程。
- 建议在 README 增加“5 分钟自检”段落：创建示例规则 -> 触发 Bash/Edit/Stop 各一次 -> 验证返回行为，降低首次接入失败率。
