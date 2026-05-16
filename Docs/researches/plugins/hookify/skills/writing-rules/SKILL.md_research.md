# FILE `plugins/hookify/skills/writing-rules/SKILL.md` 研究文档

## 场景与职责

`plugins/hookify/skills/writing-rules/SKILL.md` 是 Hookify 的规则 DSL 规范文件。它不是执行器代码，但决定了 `/hookify` 生态中“如何写规则”以及运行时“如何解释规则”。

在插件链路中的定位：

1. 上游调用方（谁显式依赖此 skill）
- `/hookify` 主命令要求第一步加载该 skill，再生成 `.claude/hookify.*.local.md`：`plugins/hookify/commands/hookify.md:9,84-137`
- `/hookify:list` 要求先加载该 skill 再读取规则：`plugins/hookify/commands/list.md:8,14-23`
- `/hookify:configure` 要求先加载该 skill 再做启停：`plugins/hookify/commands/configure.md:8,16-31`

2. 被调用方（skill 产出的规则最终由谁消费）
- hook 入口：`plugins/hookify/hooks/hooks.json:4-47`
- 规则加载：`plugins/hookify/core/config_loader.py:198-241`
- 规则匹配与阻断/告警：`plugins/hookify/core/rule_engine.py:35-94`
- 执行器：`plugins/hookify/hooks/pretooluse.py:35-59`、`posttooluse.py:30-53`、`stop.py:30-44`、`userpromptsubmit.py:30-44`

3. 平台发现关系
- `hookify` 插件由 marketplace 注册：`.claude-plugin/marketplace.json:84-92`
- 插件能力总览声明包含该 skill：`plugins/README.md:22`
- Claude Code 的 skill 发现机制：扫描 `skills/*/SKILL.md` 并按 description 命中触发：`plugins/plugin-dev/skills/plugin-structure/SKILL.md:343-347`、`plugins/plugin-dev/skills/plugin-structure/references/component-patterns.md:25`

结论：该文件是“规则作者协议层”，职责是把自然语言治理意图收敛为稳定 DSL；执行正确性由 core/hooks 保证。

## 功能点目的

### 1. 定义可落盘规则文件契约
- 明确规则文件形态为 Markdown + YAML frontmatter，存放路径为 `.claude/hookify.{rule-name}.local.md`：`plugins/hookify/skills/writing-rules/SKILL.md:11,285-287,308`
- 目的：让规则与项目绑定、可版本外管理（local）、并可被 runtime 动态加载。

### 2. 统一字段语义
- 规范 `name/enabled/event/action/pattern/conditions`：`plugins/hookify/skills/writing-rules/SKILL.md:29-98`
- 目的：减少不同命令、不同 agent 生成规则时的语义漂移。

### 3. 覆盖事件模型与常见字段
- 事件类型：`bash/file/stop/prompt/all`：`plugins/hookify/skills/writing-rules/SKILL.md:41-47,361-366`
- 字段示例：`command/file_path/new_text/old_text/content/user_prompt`：`plugins/hookify/skills/writing-rules/SKILL.md:85-88,368-371`
- 目的：让规则写作者知道在不同事件下应该匹配什么数据。

### 4. 提供 regex 与规则调优方法
- regex 写法与陷阱：`plugins/hookify/skills/writing-rules/SKILL.md:222-281`
- 本地测试命令：`python3 -c "import re; ..."`：`plugins/hookify/skills/writing-rules/SKILL.md:254-260`
- 目的：降低误报/漏报，提升规则可维护性。

### 5. 提供规则生命周期操作规范
- 创建、迭代、禁用、删除流程：`plugins/hookify/skills/writing-rules/SKILL.md:300-320`
- 示例来源目录：`${CLAUDE_PLUGIN_ROOT}/examples/`：`plugins/hookify/skills/writing-rules/SKILL.md:322-327`
- 目的：把“写规则”变成稳定可复用流程，而非一次性 prompt。

## 具体技术实现（关键流程/数据结构/协议/命令）

### A. 关键流程

1. 触发与加载
- 插件启用后，平台发现 `skills/writing-rules/SKILL.md` 元数据并在命中 description 时加载正文：`plugins/plugin-dev/skills/plugin-structure/SKILL.md:343-347`、`plugins/plugin-dev/skills/skill-development/SKILL.md:44,271-276`

2. 规则创建/管理
- `/hookify` 根据 skill 语法创建规则文件：`plugins/hookify/commands/hookify.md:84-137`
- `/hookify:list`、`/hookify:configure` 依此语法读取/编辑 frontmatter：`plugins/hookify/commands/list.md:19-23`、`plugins/hookify/commands/configure.md:28-90`

3. 运行时执行
- 事件触发 -> `hooks/*.py` 读取 stdin JSON -> `load_rules(event)` -> `RuleEngine.evaluate_rules(...)` -> stdout JSON：
  - `plugins/hookify/hooks/pretooluse.py:39-59`
  - `plugins/hookify/core/config_loader.py:209-227`
  - `plugins/hookify/core/rule_engine.py:35-94`

### B. 关键数据结构与 DSL 映射

1. SKILL 中的规则 DSL
- simple：`pattern`
- advanced：`conditions[]`（`field/operator/pattern`）
- message：frontmatter 后正文：`plugins/hookify/skills/writing-rules/SKILL.md:53-98,100-125`

2. Python 内部对象
- `Condition(field, operator, pattern)`：`plugins/hookify/core/config_loader.py:15-29`
- `Rule(name, enabled, event, pattern, conditions, action, tool_matcher, message)`：`plugins/hookify/core/config_loader.py:32-43`

3. simple pattern 的自动映射
- `event=bash` -> `field=command`
- `event=file` -> `field=new_text`
- 其他事件 -> `field=content`
- 映射实现：`plugins/hookify/core/config_loader.py:57-73`

### C. 协议（Hook 输入/输出）

1. 输入协议（stdin JSON）
- 公共字段与事件字段：`hook_event_name/tool_name/tool_input/transcript_path/user_prompt/reason`：`plugins/plugin-dev/skills/hook-development/SKILL.md:300-319`

2. 输出协议（stdout JSON）
- `warn`：返回 `systemMessage`
- `block` + `PreToolUse/PostToolUse`：`hookSpecificOutput.permissionDecision = "deny"`
- `block` + `Stop`：`decision = "block"`
- 实现：`plugins/hookify/core/rule_engine.py:60-91`

### D. 命令与脚本实证

本次研究执行了最小实证：

1. 直接执行 hook 执行器（无规则场景）
- `CLAUDE_PLUGIN_ROOT=... python3 plugins/hookify/hooks/pretooluse.py < sample.json`
- `CLAUDE_PLUGIN_ROOT=... python3 plugins/hookify/hooks/stop.py < sample.json`
- `CLAUDE_PLUGIN_ROOT=... python3 plugins/hookify/hooks/userpromptsubmit.py < sample.json`
- 结果均为 `{}`，与“无匹配规则放行”语义一致（对应 `rule_engine.py:94`）。

2. 规则语义验证（Python 直连引擎）
- 验证 `event: stop + pattern: .*` 会被映射为 `field=content`，在 Stop 输入通常不命中。
- 验证 `event: file + pattern` 默认匹配 `new_text`，对 `Write.content` 不一定命中。
- 验证 `not_contains` 为字面子串判断，不是正则否定。

3. 开发脚本兼容性验证
- 执行 `plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh plugins/hookify/hooks/hooks.json`
- 发现该脚本按“root 直接是事件键”假设校验，而 `hookify/hooks/hooks.json` 使用 `{description, hooks:{...}}` 包装，导致脚本报错（jq 索引字符串失败）。

## 关键代码路径与文件引用

目标文件：
- `plugins/hookify/skills/writing-rules/SKILL.md:1-374`

上游调用方：
- `plugins/hookify/commands/hookify.md:9,84-137`
- `plugins/hookify/commands/list.md:8,14-23`
- `plugins/hookify/commands/configure.md:8,16-31`
- `plugins/hookify/README.md:71-260`

下游被调用方：
- `plugins/hookify/hooks/hooks.json:4-47`
- `plugins/hookify/hooks/pretooluse.py:35-59`
- `plugins/hookify/hooks/posttooluse.py:30-53`
- `plugins/hookify/hooks/stop.py:30-44`
- `plugins/hookify/hooks/userpromptsubmit.py:30-44`
- `plugins/hookify/core/config_loader.py:15-84,198-275`
- `plugins/hookify/core/rule_engine.py:35-180,182-274`

配置与插件发现：
- `plugins/hookify/.claude-plugin/plugin.json:1-9`
- `.claude-plugin/marketplace.json:84-92`
- `plugins/plugin-dev/skills/plugin-structure/SKILL.md:343-347`

测试与脚本上下文：
- `plugins/hookify` 下无独立 `tests/`；仅 core 文件中有 `__main__` 手工验证片段：
  - `plugins/hookify/core/config_loader.py:277-297`
  - `plugins/hookify/core/rule_engine.py:276-313`
- 通用 hook 校验脚本与样本生成脚本：
  - `plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh:41-75`
  - `plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh:25-100`

文档依赖：
- `plugins/hookify/README.md:71-260`
- `plugins/hookify/commands/help.md:16-159`
- `plugins/hookify/examples/*.local.md`

## 依赖与外部交互

### 1. 内部依赖
- 依赖 Claude Code skill 发现与触发机制（基于 frontmatter `description`）：`plugins/plugin-dev/skills/plugin-structure/references/component-patterns.md:25`
- 依赖 hookify 命令层优先加载该 skill，确保输出 DSL 与 runtime 一致：`plugins/hookify/commands/hookify.md:9`
- 依赖 core 模块解释 frontmatter 并执行匹配：`plugins/hookify/core/config_loader.py:45-84`、`plugins/hookify/core/rule_engine.py:144-180`

### 2. 外部交互面
- 与宿主 Hook 协议交互：stdin 输入 JSON，stdout 输出 JSON。
- 与文件系统交互：读取项目 `.claude/hookify.*.local.md`，Stop 场景可读取 `transcript_path` 文件：`plugins/hookify/core/config_loader.py:210`、`plugins/hookify/core/rule_engine.py:207-225`
- 不依赖网络与第三方服务；Python 标准库实现：`plugins/hookify/README.md:300-301`

### 3. 配置约束
- 规则路径约定：`.claude/hookify.{name}.local.md`
- hook 路由约定：`hooks/hooks.json` 指向四个 Python 执行器
- 环境变量约定：`CLAUDE_PLUGIN_ROOT` 用于导入 `hookify.core`：`plugins/hookify/hooks/pretooluse.py:14-23`

## 风险、边界与改进建议

### 风险

1. `stop/prompt` 的 simple `pattern` 有失配风险
- 技术原因：`Rule.from_dict` 对非 `bash/file` 默认映射 `field=content`：`plugins/hookify/core/config_loader.py:67`
- 运行现实：Stop 常用字段是 `reason/transcript`，Prompt 常用 `user_prompt`：`plugins/hookify/core/rule_engine.py:205-229`
- 影响：skill 文档中的 `event: stop + pattern: .*` 写法在不少真实输入中不触发。

2. `file` simple `pattern` 对 Write 场景覆盖不足
- 默认映射 `new_text`：`plugins/hookify/core/config_loader.py:64-65`
- Write 常用 `content`，`new_text` 为空时会漏报：`plugins/hookify/core/rule_engine.py:235-240`

3. `not_contains` 易被误用为“正则否定”
- 实际实现是字面子串：`plugins/hookify/core/rule_engine.py:172-173`
- 示例中常见 `npm test|pytest|cargo test` 会被当作完整字符串而非 OR。

4. 脚本校验口径与 hookify 实际配置不一致
- `validate-hook-schema.sh` 假设根层是事件键；hookify 采用 `{"description","hooks"}` 包装。
- 会导致“校验脚本不可直接用于 hookify 当前 hooks.json”的误导。

5. 文档能力与运行能力存在轻度漂移
- `Rule` 支持 `tool_matcher`，但 `writing-rules/SKILL.md` 未介绍该字段：`plugins/hookify/core/config_loader.py:41-42,82`
- 结果是高级过滤能力不可见。

### 边界

1. 本文件是规则写作规范，不承担 hook 执行与进程控制。
2. 不直接创建/读取规则文件，创建行为属于命令层。
3. 不包含自动化测试代码；质量主要依赖文档约束与手工验证。

### 改进建议

1. 修正 simple pattern 的事件默认映射
- `stop -> transcript`（或 `reason`），`prompt -> user_prompt`，并在 `SKILL.md` 同步说明。

2. 强化 file 场景建议
- 在 `SKILL.md` 中明确：对于 `Write` 优先用 `conditions` 且匹配 `content`。

3. 新增否定正则操作符
- 增加 `not_regex_match`，避免 `not_contains` 被误解。

4. 文档补齐 `tool_matcher`
- 在 `SKILL.md` 新增“按 tool_name 过滤”章节，示例 `Edit|Write|MultiEdit`。

5. 增加最小回归测试
- 建议新增 hookify 级 smoke：
  - Bash block 规则命中
  - Write+content 规则命中
  - Stop transcript 规则命中
  - not_contains 与 regex 行为区分

6. 给开发脚本加 hookify 兼容模式
- `validate-hook-schema.sh` 支持解析 root 包装结构（`hooks` 字段）或为 hookify 提供专用 validator。
