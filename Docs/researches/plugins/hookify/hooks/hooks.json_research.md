# FILE `plugins/hookify/hooks/hooks.json` 研究文档

## 场景与职责

`plugins/hookify/hooks/hooks.json` 是 Hookify 插件的 Hook 路由清单，职责是把 Claude Code 事件映射到插件执行脚本。

该文件定义了 4 类事件：`PreToolUse`、`PostToolUse`、`Stop`、`UserPromptSubmit`，分别路由到 `hooks/*.py` 执行器，实现“规则在工具前后、会话结束、用户提问时统一触发”。

## 功能点目的

1. 统一事件入口。
- 将 Claude Code 多个生命周期事件集中注册到 Hookify，避免分散配置。

2. 命令型 hook 固化。
- 每个事件都使用 `type: command`，通过 Python 脚本执行确定性逻辑。

3. 跨安装路径可移植。
- 使用 `${CLAUDE_PLUGIN_ROOT}` 拼接脚本路径，避免硬编码绝对路径。

4. 基础超时控制。
- 每个 hook 显式设置 `timeout: 10`，控制单次执行最长时间。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 配置结构

文件采用插件 hooks wrapper 格式：

- 顶层 `description`
- 顶层 `hooks` 对象
- `hooks.<Event>` 为数组
- 每个数组项包含内部 `hooks` 数组
- 每个内部 hook 声明 `type/command/timeout`

对应实现：`plugins/hookify/hooks/hooks.json:1-49`。

### 2) 事件到命令映射

- `PreToolUse` -> `python3 ${CLAUDE_PLUGIN_ROOT}/hooks/pretooluse.py`（`hooks.json:4-10`）
- `PostToolUse` -> `python3 ${CLAUDE_PLUGIN_ROOT}/hooks/posttooluse.py`（`hooks.json:15-21`）
- `Stop` -> `python3 ${CLAUDE_PLUGIN_ROOT}/hooks/stop.py`（`hooks.json:26-32`）
- `UserPromptSubmit` -> `python3 ${CLAUDE_PLUGIN_ROOT}/hooks/userpromptsubmit.py`（`hooks.json:37-43`）

### 3) 运行链路

1. 插件 manifest 未覆写 `hooks` 字段（`plugins/hookify/.claude-plugin/plugin.json:1-9`）。
2. 按插件规范默认读取 `./hooks/hooks.json`（`manifest-reference.md:261-263`）。
3. 事件触发后执行对应命令脚本。
4. 脚本读取 stdin JSON，调用 `hookify.core` 完成规则判定并输出 JSON。

### 4) 与工具链脚本的格式兼容性

仓库 `validate-hook-schema.sh` 按“顶层即事件键”校验（`validate-hook-schema.sh:43-66`），与当前插件 wrapper 结构不一致。

对本文件实测结果：

- 警告 `Unknown event type: description/hooks`
- 随后 `jq: Cannot index string with number`
- 退出码 `5`

这说明校验脚本当前不能直接用于该文件格式。

## 关键代码路径与文件引用

- 目标配置：`plugins/hookify/hooks/hooks.json:1-49`
- 对应执行器：
  - `plugins/hookify/hooks/pretooluse.py:1-74`
  - `plugins/hookify/hooks/posttooluse.py:1-66`
  - `plugins/hookify/hooks/stop.py:1-59`
  - `plugins/hookify/hooks/userpromptsubmit.py:1-58`
- 规则加载与求值：
  - `plugins/hookify/core/config_loader.py:198-241`
  - `plugins/hookify/core/rule_engine.py:35-94`
- 默认 hooks 路径规范：`plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:261-263`
- 插件 hooks wrapper 规范：`plugins/plugin-dev/skills/hook-development/SKILL.md:62-80`
- 当前不兼容校验脚本：`plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh:41-86`

## 依赖与外部交互

1. Claude Code Hook runtime：负责加载本配置并在事件上执行命令。
2. 运行时环境变量：`${CLAUDE_PLUGIN_ROOT}` 由宿主注入用于路径解析。
3. 外部命令依赖：`python3` 必须可用。
4. 下游脚本依赖：`hooks/*.py` 再依赖 `hookify.core` 与 `.claude/hookify.*.local.md` 规则文件。

## 风险、边界与改进建议

1. `PreToolUse`/`PostToolUse` 未设置 `matcher`。
- 风险：所有工具事件都会触发脚本，增加无效调用。
- 建议：增加 `matcher: "Bash|Edit|Write|MultiEdit"` 或按事件拆分更精细匹配。

2. 校验脚本与插件格式不兼容。
- 风险：研发流程中容易把“脚本报错”误判为配置错误。
- 建议：扩展 `validate-hook-schema.sh` 支持 wrapper 格式（先下钻 `.hooks`）。

3. 超时固定 10 秒。
- 风险：规则复杂、Stop 场景读取 transcript 时可能超时；同时缺少按事件差异化配置。
- 建议：按事件设置不同 timeout（例如 Stop 稍高）。

4. 事件覆盖边界。
- 现状：未注册 `SessionStart/SessionEnd/SubagentStop`。
- 建议：若需要“会话级策略”或“子代理收尾约束”，可补充对应事件配置。
