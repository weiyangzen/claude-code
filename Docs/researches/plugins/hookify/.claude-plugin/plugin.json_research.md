# plugins/hookify/.claude-plugin/plugin.json 研究

## 场景与职责

`plugins/hookify/.claude-plugin/plugin.json` 是 `hookify` 插件的 manifest 入口文件，位于 Claude Code 约定的必需路径 `.claude-plugin/plugin.json`，用于让插件被识别与装配，而不是直接承载 Hook 规则执行逻辑（`plugins/hookify/.claude-plugin/plugin.json:1-9`，`plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:7-10`）。

在 `hookify` 场景里，它承担三层职责：

1. 插件身份声明
- 提供 `name/version/description/author` 基础元数据（`plugins/hookify/.claude-plugin/plugin.json:2-8`）。

2. 运行时发现链路入口
- 按规范，Claude Code 会先识别 manifest，再进入默认组件发现（`plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:352-371`）。
- `hookify` 的可执行能力随后来自 `commands/`、`agents/`、`skills/`、`hooks/hooks.json`（`plugins/README.md:49-60`，`plugins/hookify/hooks/hooks.json:1-49`）。

3. 分发与展示锚点
- marketplace 将 `hookify` 注册到 `source: ./plugins/hookify`，并复写更完整的描述文本，形成“市场索引 -> 插件目录 -> manifest”的装配路径（`.claude-plugin/marketplace.json:84-93`）。

## 功能点目的

### 1) `name`
- `name: "hookify"` 用于插件唯一识别与冲突检测（`plugins/hookify/.claude-plugin/plugin.json:2`，`plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:21-24`）。

### 2) `version`
- `version: "0.1.0"` 提供语义版本信息，支持发布/升级管理（`plugins/hookify/.claude-plugin/plugin.json:3`，`plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:42-63`）。
- 与 marketplace 保持一致（`.claude-plugin/marketplace.json:86`）。

### 3) `description`
- 说明插件目标是“通过对话模式分析来防止不希望行为”（`plugins/hookify/.claude-plugin/plugin.json:4`）。
- 实际用户向能力在 README 被展开为 `/hookify` 系命令 + 动态 `.local.md` 规则体系（`plugins/hookify/README.md:39-69`）。

### 4) `author`
- 作者归属与联系信息（`plugins/hookify/.claude-plugin/plugin.json:5-8`）。
- 与 marketplace 作者字段一致（`.claude-plugin/marketplace.json:87-90`）。

### 5) “不配置即采用默认”
- 当前 manifest 不含 `commands/agents/hooks/mcpServers` 路径字段，因此依赖默认扫描：`./commands`、`./agents`、`./skills`、`./hooks/hooks.json`（`plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:356-361`）。
- 这与 `hookify` 当前目录结构一致（`plugins/hookify/commands/hookify.md:1-5`，`plugins/hookify/agents/conversation-analyzer.md:1-7`，`plugins/hookify/skills/writing-rules/SKILL.md:1-5`，`plugins/hookify/hooks/hooks.json:1-49`）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### A. 关键流程

1. 上游注册与定位
- marketplace 声明插件源目录 `./plugins/hookify`（`.claude-plugin/marketplace.json:91`）。

2. manifest 识别
- 运行时按规范读取 `.claude-plugin/plugin.json`（`plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:7-10`）。

3. 默认组件发现
- 因 manifest 未重载路径，走默认目录发现（`plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:356-371`）。

4. 下游能力装配
- 命令层：`/hookify`、`/hookify:list`、`/hookify:configure`、`/hookify:help`（`plugins/hookify/commands/hookify.md:1-5`，`plugins/hookify/commands/list.md:1-4`，`plugins/hookify/commands/configure.md:1-4`，`plugins/hookify/commands/help.md:1-4`）。
- Agent：`conversation-analyzer` 用于无参 `/hookify` 的行为抽取（`plugins/hookify/agents/conversation-analyzer.md:2-7,171-176`）。
- Skill：`writing-rules` 约束规则格式与字段语义（`plugins/hookify/skills/writing-rules/SKILL.md:2-4,29-57`）。
- Hook 运行时：`hooks/hooks.json` 将 4 个事件绑定到 Python command hook（`plugins/hookify/hooks/hooks.json:4-47`）。

5. 规则执行路径
- 每个 hook 脚本从 stdin 读取事件 JSON，加载 `.claude/hookify.*.local.md` 规则并评估（`plugins/hookify/hooks/pretooluse.py:38-57`，`plugins/hookify/core/config_loader.py:198-241`，`plugins/hookify/core/rule_engine.py:35-95`）。

简化链路：

`marketplace(source)` -> `plugins/hookify/.claude-plugin/plugin.json` -> 默认发现 commands/agents/skills/hooks -> `hooks/hooks.json` 调 Python 脚本 -> 读取 `.claude/hookify.*.local.md` -> 规则匹配后返回 Hook 协议 JSON。

### B. 关键数据结构

1. 插件 manifest（目标对象）
```json
{
  "name": "hookify",
  "version": "0.1.0",
  "description": "Easily create hooks to prevent unwanted behaviors by analyzing conversation patterns",
  "author": {
    "name": "Daisy Hollman",
    "email": "daisy@anthropic.com"
  }
}
```
来源：`plugins/hookify/.claude-plugin/plugin.json:1-9`

2. Hookify 规则模型
- `Condition`: `field/operator/pattern`
- `Rule`: `name/enabled/event/pattern/conditions/action/tool_matcher/message`
（`plugins/hookify/core/config_loader.py:15-43`）。

3. 规则文件载入目标
- 规则文件固定从项目工作目录下 `.claude/hookify.*.local.md` 扫描（`plugins/hookify/core/config_loader.py:209-211`）。

### C. 协议与命令

1. Hook 配置协议
- `hookify` 使用插件 wrapper 格式：`{"description": ..., "hooks": {...}}`（`plugins/hookify/hooks/hooks.json:1-4`）。
- 事件覆盖 `PreToolUse/PostToolUse/Stop/UserPromptSubmit`，每个事件调用 `python3 ${CLAUDE_PLUGIN_ROOT}/hooks/*.py`（`plugins/hookify/hooks/hooks.json:4-47`）。

2. Hook 输入协议
- hook 从 stdin 接收 JSON（`plugins/hookify/hooks/pretooluse.py:38-39`）。
- 规范定义了通用字段 `hook_event_name` 与事件特定字段（`plugins/plugin-dev/skills/hook-development/SKILL.md:300-319`）。

3. Hook 输出协议
- `RuleEngine` 根据事件返回不同结构：
  - Stop：`decision: block` + `reason`（`plugins/hookify/core/rule_engine.py:66-71`）
  - Pre/PostToolUse：`hookSpecificOutput.permissionDecision: deny`（`plugins/hookify/core/rule_engine.py:72-79`）
  - Warn：`systemMessage`（`plugins/hookify/core/rule_engine.py:86-91`）

4. 命令协议
- `/hookify` 指导模型收集行为、询问用户、生成 `.claude/hookify.*.local.md`（`plugins/hookify/commands/hookify.md:17-31,82-99,126-137`）。
- `/hookify:configure`、`/hookify:list` 以 Glob + Read/Edit 管理规则启停（`plugins/hookify/commands/configure.md:14-19,73-90`，`plugins/hookify/commands/list.md:14-23`）。

### D. 测试与脚本现状（针对本对象相关链路）

1. 插件内自动化测试缺失
- `plugins/hookify` 目录无 `test/spec` 文件，主要由文档与示例驱动（`find plugins/hookify -type f | sort` 结果）。

2. 可复用脚本来自 `plugin-dev`
- `validate-hook-schema.sh`、`test-hook.sh`、`hook-linter.sh` 是推荐验证工具（`plugins/plugin-dev/skills/hook-development/scripts/README.md:5-60,62-90`）。

3. 与 `hookify` 实际格式的兼容性问题
- 实测执行：
  - `bash plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh plugins/hookify/hooks/hooks.json`
  - 结果：`jq: Cannot index string with number`，退出码 `5`。
- 原因是脚本按顶层事件遍历（`jq -r 'keys[]'`）并要求 `event[i].matcher`（`plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh:43,65-73`），而 `hookify` 文件是 wrapper 结构且事件层未写 matcher（`plugins/hookify/hooks/hooks.json:1-6`）。

## 关键代码路径与文件引用

### 目标对象
- `plugins/hookify/.claude-plugin/plugin.json:1-9`

### 调用方（上游）
- `.claude-plugin/marketplace.json:84-93`（hookify 注册与 source）
- `plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:7-10`（manifest 必须路径）
- `plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:352-371`（默认发现与合并行为）

### 被调用方（下游）
- `plugins/hookify/commands/hookify.md:1-5`
- `plugins/hookify/commands/list.md:1-4`
- `plugins/hookify/commands/configure.md:1-4`
- `plugins/hookify/commands/help.md:1-4`
- `plugins/hookify/agents/conversation-analyzer.md:1-7`
- `plugins/hookify/skills/writing-rules/SKILL.md:1-5`
- `plugins/hookify/hooks/hooks.json:1-49`
- `plugins/hookify/hooks/pretooluse.py:35-70`
- `plugins/hookify/hooks/posttooluse.py:30-62`
- `plugins/hookify/hooks/stop.py:30-55`
- `plugins/hookify/hooks/userpromptsubmit.py:30-54`
- `plugins/hookify/core/config_loader.py:198-274`
- `plugins/hookify/core/rule_engine.py:35-274`

### 配置、测试、脚本、文档上下文
- 配置：
  - `plugins/hookify/.claude-plugin/plugin.json:1-9`
  - `plugins/hookify/hooks/hooks.json:1-49`
  - `.claude/hookify.*.local.md` 规则文件约定（`plugins/hookify/core/config_loader.py:209-211`）
- 测试/脚本：
  - `plugins/plugin-dev/skills/hook-development/scripts/README.md:5-60,92-128`
  - `plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh:41-73`
- 文档：
  - `plugins/hookify/README.md:39-69,289-301`
  - `plugins/README.md:22,49-70`

## 依赖与外部交互

### 仓库内依赖

1. marketplace 目录映射依赖
- `source` 必须正确指向插件目录，否则 manifest 不会被装配（`.claude-plugin/marketplace.json:91`）。

2. 默认目录约定依赖
- 因未配置自定义路径，实际依赖默认发现约定（`plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:356-361`）。

3. Hookify 内部 Python 模块依赖
- Hook 脚本依赖 `hookify.core.config_loader` 和 `hookify.core.rule_engine`，并通过 `CLAUDE_PLUGIN_ROOT` 动态补 `sys.path`（`plugins/hookify/hooks/pretooluse.py:12-27`）。

### 外部交互

1. 运行时进程与环境
- 通过 command hook 启动 `python3` 子进程（`plugins/hookify/hooks/hooks.json:9,20,31,42`）。
- 依赖环境变量 `CLAUDE_PLUGIN_ROOT` 做路径可移植（`plugins/hookify/hooks/pretooluse.py:14-23`，`plugins/plugin-dev/skills/hook-development/SKILL.md:326-337`）。

2. 文件系统交互
- 扫描并读取项目目录 `.claude/hookify.*.local.md`（`plugins/hookify/core/config_loader.py:209-253`）。
- `stop` 规则可读取 `transcript_path` 文件内容进行匹配（`plugins/hookify/core/rule_engine.py:207-225`）。

3. 网络与第三方依赖
- `hookify` 本身无网络调用；README 亦声明仅使用 Python 标准库（`plugins/hookify/README.md:298-301`）。

## 风险、边界与改进建议

### 风险

1. 元数据多源漂移
- `plugin.json` 与 marketplace 同时维护 `description/version/author`，长期存在不一致风险（`plugins/hookify/.claude-plugin/plugin.json:3-8` vs `.claude-plugin/marketplace.json:85-90`）。

2. 脚本校验与实际格式不一致
- `validate-hook-schema.sh` 当前不兼容 wrapper 结构，无法直接作为 `hookify/hooks/hooks.json` 的可靠校验器（`plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh:43,65-73`，`plugins/hookify/hooks/hooks.json:1-4`）。

3. 规则解析健壮性边界
- `extract_frontmatter` 是手写 YAML 解析器，支持有限，复杂 YAML（多层嵌套、特殊字符）存在解析偏差风险（`plugins/hookify/core/config_loader.py:87-195`）。

4. 运行错误可见性风险
- hook 脚本在异常场景统一 `exit 0`，虽然保证不中断流程，但也可能掩盖规则失效（`plugins/hookify/hooks/pretooluse.py:68-70`，`plugins/hookify/hooks/stop.py:53-55`）。

5. 兼容性变更风险
- Changelog 显示 `plugin.json` 字段兼容历史上发生过修复，意味着 manifest 消费侧规则可能演进（`CHANGELOG.md:233`）。

### 边界

1. 本对象只负责插件元信息与发现入口，不执行 Hook 业务判断。
2. 真正策略在 `hooks/*.py + core/* + .claude/hookify.*.local.md`。
3. 插件行为强依赖 Claude Code 运行时 Hook 协议；仓库内不含该运行时实现。

### 改进建议

1. 增加一致性校验
- 在 CI 增加 manifest 与 marketplace 关键字段比对（至少 `name/version/author/description`）。

2. 修复 hooks schema 工具兼容性
- 让 `validate-hook-schema.sh` 同时支持 wrapper（`{"hooks": ...}`）和 direct 两种格式，避免对官方插件误报。

3. 增加 hookify 冒烟测试
- 覆盖：manifest 可解析、`hooks/hooks.json` 可加载、样例规则在 `PreToolUse/Stop` 返回协议 JSON。

4. 结构化错误观测
- 在保持 `exit 0` 的同时，把失败原因输出到可检索日志通道（而非仅 `systemMessage`），提高排障效率。

5. 规范化规则解析
- 中长期考虑以成熟 YAML 解析方案替换手写 parser，降低边界输入导致的解析歧义。
