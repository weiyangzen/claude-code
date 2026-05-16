# DIR `plugins/hookify/.claude-plugin` 研究文档

## 场景与职责

`plugins/hookify/.claude-plugin` 目录是 Hookify 插件的**元数据入口层**，当前仅包含 `plugin.json`。

在 Claude Code 插件生命周期里，这个目录承担三类职责：

1. **插件发现入口**：插件系统在启用插件时先读取 `.claude-plugin/plugin.json`（插件结构规范中的必需路径）。
2. **身份与版本声明**：向插件系统与 marketplace 声明 `name/version/description/author`。
3. **默认组件装配锚点**：Hookify 没有在 `plugin.json` 中显式声明 `commands/agents/skills/hooks` 路径，依赖默认目录发现机制，把能力转交给 `commands/`、`agents/`、`skills/`、`hooks/hooks.json`。

这意味着该目录本身代码量很小，但对“插件能否被识别、能否被正确装配”是硬门槛。

## 功能点目的

围绕 `plugin.json` 的核心功能点与目的如下：

1. **唯一标识插件**
   - `name: hookify` 作为插件标识，支撑冲突检测与注册。
2. **声明分发版本**
   - `version: 0.1.0` 用于插件发布与兼容性沟通。
3. **声明用途说明**
   - `description` 直接表达该插件价值：通过对话模式分析与显式指令，生成/执行规则化 Hook。
4. **声明作者归属**
   - `author.name/email` 用于归属与维护联系。
5. **与 marketplace 条目对齐**
   - 顶层 `.claude-plugin/marketplace.json` 对 hookify 有单独条目；该目录中的 `plugin.json` 是对应 source（`./plugins/hookify`）内的真实插件清单。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 关键流程：从 manifest 到规则执行

1. marketplace 注册 `hookify`，`source` 指向 `./plugins/hookify`。
2. 插件系统读取 `plugins/hookify/.claude-plugin/plugin.json` 完成插件识别。
3. 因 manifest 未覆写组件路径，系统按默认路径装载：
   - `commands/`（`/hookify` 系列）
   - `agents/`（`conversation-analyzer`）
   - `skills/`（`writing-rules`）
   - `hooks/hooks.json`（四类事件 Hook）
4. 运行时事件触发 `hooks/hooks.json` 中的 command hook：
   - `PreToolUse` -> `python3 .../hooks/pretooluse.py`
   - `PostToolUse` -> `python3 .../hooks/posttooluse.py`
   - `Stop` -> `python3 .../hooks/stop.py`
   - `UserPromptSubmit` -> `python3 .../hooks/userpromptsubmit.py`
5. hook 脚本读取 stdin JSON，调用：
   - `load_rules(event=...)` 从 `.claude/hookify.*.local.md` 加载规则
   - `RuleEngine.evaluate_rules(...)` 计算命中并输出协议 JSON

### 2) 关键数据结构

1. **manifest 结构（本目录核心）**
   - `name`, `version`, `description`, `author`
2. **规则结构（被本目录间接激活）**
   - `Rule`：`name/enabled/event/pattern/conditions/action/tool_matcher/message`
   - `Condition`：`field/operator/pattern`
3. **Hook 协议输出结构**
   - Pre/Post 拒绝：`hookSpecificOutput.permissionDecision = "deny"` + `systemMessage`
   - Stop 阻断：`decision = "block"` + `reason/systemMessage`
   - 仅提醒：`systemMessage`

### 3) 关键协议与命令

1. **插件元数据协议**：`.claude-plugin/plugin.json`（必需）
2. **Hook 配置协议**：`hooks/hooks.json` 使用插件包装格式 `{"description":..., "hooks": {...}}`
3. **规则文件协议**：`.claude/hookify.*.local.md`（YAML frontmatter + markdown message）
4. **常用命令入口**：
   - `/hookify`（创建规则）
   - `/hookify:list`（枚举规则）
   - `/hookify:configure`（启停规则）
   - `/hookify:help`（解释机制）
5. **手工验证命令（本次研究实测）**：
   - `python3 -m py_compile plugins/hookify/core/*.py plugins/hookify/hooks/*.py` -> 通过
   - `bash plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh ...pretooluse.py ...` -> 输出 `{}`，exit 0
   - `bash plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh plugins/hookify/hooks/hooks.json` -> 对插件包装格式不兼容（报 jq 索引错误）

## 关键代码路径与文件引用

### A. 目标目录（直接研究对象）

- `plugins/hookify/.claude-plugin/plugin.json`

### B. 直接调用方/装配上下文

- `.claude-plugin/marketplace.json`（hookify 条目与 source）
- `plugins/README.md`（插件能力目录与结构约定）
- `plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md`（manifest 必需路径、默认组件路径）
- `plugins/plugin-dev/skills/plugin-structure/references/component-patterns.md`（启动阶段读取 plugin.json 的生命周期说明）

### C. 被调用方（manifest 激活的实际能力）

- `plugins/hookify/hooks/hooks.json`
- `plugins/hookify/hooks/pretooluse.py`
- `plugins/hookify/hooks/posttooluse.py`
- `plugins/hookify/hooks/stop.py`
- `plugins/hookify/hooks/userpromptsubmit.py`
- `plugins/hookify/core/config_loader.py`
- `plugins/hookify/core/rule_engine.py`

### D. 配置与规则生产链路

- `plugins/hookify/commands/hookify.md`
- `plugins/hookify/commands/list.md`
- `plugins/hookify/commands/configure.md`
- `plugins/hookify/commands/help.md`
- `plugins/hookify/agents/conversation-analyzer.md`
- `plugins/hookify/skills/writing-rules/SKILL.md`
- `plugins/hookify/examples/*.local.md`

### E. 测试/脚本/文档上下文

- `plugins/plugin-dev/skills/hook-development/SKILL.md`
- `plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh`
- `plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh`
- `plugins/plugin-dev/skills/hook-development/scripts/README.md`

说明：`plugins/hookify` 自身没有独立 `tests/` 或自动化测试脚本目录。

## 依赖与外部交互

1. **运行时依赖**
   - Python 3（hook 命令显式调用 `python3`）
   - Python 标准库（`json/re/glob/dataclasses/...`）
2. **环境变量依赖**
   - `CLAUDE_PLUGIN_ROOT`：hook 脚本通过它注入 `sys.path` 并定位自身目录
   - `CLAUDE_PROJECT_DIR`（由平台提供，测试脚本也会注入）
3. **文件系统交互**
   - 读取项目根 `.claude/hookify.*.local.md`
   - Stop 规则可读取 `transcript_path`
4. **平台协议交互**
   - stdin 接收 Hook 事件 JSON
   - stdout 输出结构化 JSON 返回给 Claude Code
5. **分发交互**
   - marketplace 条目 + plugin manifest 共同决定安装发现路径

## 风险、边界与改进建议

1. **风险：manifest 极简，缺少扩展元数据**
   - 现状：仅 `name/version/description/author`，无 `homepage/repository/license/keywords`。
   - 影响：分发展示与可维护信息密度偏低。
   - 建议：补齐非必需元数据，提高可追踪性。

2. **风险：规则解析器是手写 YAML 子集**
   - 现状：`extract_frontmatter` 不等价完整 YAML 语义。
   - 影响：复杂 frontmatter（缩进/转义/多行）存在兼容边界。
   - 建议：引入成熟 YAML 解析器或增加严格 schema 校验与错误提示。

3. **风险：规则扫描路径绑定当前工作目录**
   - 现状：固定扫描 `.claude/hookify.*.local.md`。
   - 影响：多仓库/非标准 cwd 下，规则可能“看起来存在但未加载”。
   - 建议：结合 `CLAUDE_PROJECT_DIR` 做绝对路径解析，或在错误消息中输出当前扫描根路径。

4. **风险：通用校验脚本与插件 hooks 包装格式不一致**
   - 现状：`validate-hook-schema.sh` 按“顶层事件键”读取，而 hookify 使用 `{"hooks": {...}}` 包装。
   - 实测：校验脚本对 hookify 直接报错并提前退出（jq 索引 string）。
   - 建议：升级脚本同时兼容“插件包装格式”和“settings 直写格式”，并将 matcher 设为可选。

5. **风险：测试覆盖不足**
   - 现状：hookify 无专属自动化测试。
   - 影响：规则解析、跨事件行为、错误恢复主要依赖手工回归。
   - 建议：增加最小 smoke tests（pre/post/stop/prompt 各一例）和规则文件解析回归样例。

6. **边界说明：本目录并不直接承载业务逻辑**
   - `.claude-plugin` 仅负责声明与发现；真正逻辑在 `hooks/` 与 `core/`。
   - 因此该目录变更虽小，但会放大影响到整个插件是否可被加载。
