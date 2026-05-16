# FILE `plugins/explanatory-output-style/hooks/hooks.json` 研究文档

## 场景与职责

`plugins/explanatory-output-style/hooks/hooks.json` 是 `explanatory-output-style` 插件的 Hook 路由入口文件，职责是把 Claude Code 的 `SessionStart` 生命周期事件绑定到具体命令处理器。

它在链路中的定位：
- 上游调用方：Claude Code 插件运行时加载插件并读取 Hook 配置（manifest 默认 hook 路径为 `./hooks/hooks.json`）。
- 下游被调用方：`plugins/explanatory-output-style/hooks-handlers/session-start.sh`。
- 业务职责边界：只做事件到命令的声明，不做输入解析、不做权限决策、不直接读写工程文件。

关键证据：
- 目标文件仅声明 `SessionStart` 与 command hook：`plugins/explanatory-output-style/hooks/hooks.json:3-13`。
- manifest 文档声明 hooks 默认文件为 `./hooks/hooks.json`：`plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:259-263`。
- 插件 README 明确该插件是“通过 SessionStart 重建已废弃 Explanatory 风格”：`plugins/explanatory-output-style/README.md:3-4,20-23`。

## 功能点目的

1. 复刻旧输出风格迁移能力
- 通过会话启动注入额外上下文，替代旧 `"outputStyle": "Explanatory"` 配置路径。
- 证据：`plugins/explanatory-output-style/README.md:42-53`。

2. 让解释型行为在每个会话自动生效
- 仅使用 `SessionStart`，意味着“会话级风格注入”，而不是工具调用级动态拦截。
- 证据：`plugins/explanatory-output-style/hooks/hooks.json:4-13`。

3. 提供可移植的插件内部命令寻址
- 命令路径使用 `${CLAUDE_PLUGIN_ROOT}`，避免硬编码绝对路径。
- 证据：`plugins/explanatory-output-style/hooks/hooks.json:9`。

4. 与同类插件保持模式一致
- `learning-output-style` 使用同构 `hooks.json` 结构，降低维护认知成本。
- 证据：`plugins/learning-output-style/hooks/hooks.json:1-15`。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 关键流程

1. 插件发现与加载
- marketplace 将该插件注册为 `source: ./plugins/explanatory-output-style`。
- 证据：`.claude-plugin/marketplace.json:51-59`。

2. Hook 配置解析
- 插件元数据文件存在但未显式覆写 `hooks` 字段，因此按默认路径读取 `hooks/hooks.json`。
- 证据：`plugins/explanatory-output-style/.claude-plugin/plugin.json:1-9`，`plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:259-263`。

3. SessionStart 事件触发
- `hooks.json` 在 `hooks.SessionStart[0].hooks[0]` 声明 `type: command` 和脚本路径。
- 证据：`plugins/explanatory-output-style/hooks/hooks.json:4-10`。

4. 命令脚本执行并返回协议 JSON
- handler 输出 `hookSpecificOutput.hookEventName=SessionStart` 与 `hookSpecificOutput.additionalContext`。
- 证据：`plugins/explanatory-output-style/hooks-handlers/session-start.sh:6-12`。

### 数据结构

目标文件采用“插件 hooks 包装格式”：

```json
{
  "description": "Explanatory mode hook that adds educational insights instructions",
  "hooks": {
    "SessionStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "${CLAUDE_PLUGIN_ROOT}/hooks-handlers/session-start.sh"
          }
        ]
      }
    ]
  }
}
```

结构解读：
- 顶层 `description`：用于说明配置意图。
- 顶层 `hooks`：真实事件容器。
- 事件键 `SessionStart`：会话启动时触发。
- 内层 `hooks[]`：可挂多个执行项；当前仅一条 command hook。

规范对照：
- 插件格式要求 wrapper（`description` + `hooks`）而非 settings 直写事件格式。
- 证据：`plugins/plugin-dev/skills/hook-development/SKILL.md:62-80,102-119`。

### 协议与命令

1. 运行时输入协议
- Hook 脚本通过 stdin 接收 JSON，`SessionStart` 输入样本包含 `session_id`、`cwd`、`hook_event_name` 等字段。
- 证据：`plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh:83-92`。

2. 运行时输出协议
- 当前下游脚本输出 `hookSpecificOutput`，通过 `additionalContext` 注入解释型指令。
- 证据：`plugins/explanatory-output-style/hooks-handlers/session-start.sh:8-10`。

3. 实测命令结果
- `bash plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh --create-sample SessionStart` 生成样本输入成功。
- `bash plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh plugins/explanatory-output-style/hooks-handlers/session-start.sh <sample>` 返回 `Exit Code: 0`，输出 JSON 可被解析。
- `bash plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh plugins/explanatory-output-style/hooks/hooks.json` 报 `Unknown event type: description/hooks` 且 `jq: Cannot index string with number`（退出码 5）。

4. 上述验证结论
- handler 链路可执行。
- 校验脚本当前实现按“顶层是事件键”遍历，与插件 wrapper 格式不兼容。
- 证据：`plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh:41-67`（直接遍历顶层键并按事件数组下标访问）。

## 关键代码路径与文件引用

核心路径：
- `plugins/explanatory-output-style/hooks/hooks.json`

直接调用链：
- `.claude-plugin/marketplace.json:51-59`（插件注册）
- `plugins/explanatory-output-style/.claude-plugin/plugin.json:1-9`（插件元数据）
- `plugins/explanatory-output-style/hooks/hooks.json:1-15`（事件->命令映射）
- `plugins/explanatory-output-style/hooks-handlers/session-start.sh:1-15`（实际执行与输出）

关联文档与规范：
- `plugins/explanatory-output-style/README.md:3-7,20-33,42-62`（行为说明、迁移、成本）
- `plugins/plugin-dev/skills/hook-development/SKILL.md:62-80,102-119,238-253`（插件 hooks 格式与 SessionStart 示例）
- `plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:259-263`（默认 hooks 路径）
- `plugins/plugin-dev/commands/create-plugin.md:257-260`（推荐验证流程：schema + test-hook + 路径检查）

测试与脚本：
- `plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh:25-100,170-191`
- `plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh:41-67`

同类对照：
- `plugins/learning-output-style/hooks/hooks.json:1-15`
- `plugins/learning-output-style/hooks-handlers/session-start.sh:1-15`

## 依赖与外部交互

运行依赖：
- Claude Code Hook 生命周期（SessionStart）。
- `bash` 执行环境（command hook）。
- 环境变量 `${CLAUDE_PLUGIN_ROOT}` 用于路径定位。

开发/验证依赖：
- `jq`（JSON 校验与解析，来自 validator/test 工具脚本）。
- `test-hook.sh`（构造样例、执行 hook）。
- `validate-hook-schema.sh`（结构校验）。

外部交互：
- 目标文件本身不发网络请求，不访问外部 API。
- 目标文件本身不直接读写仓库业务文件。
- 主要副作用是让会话默认附加更长系统上下文，增加 token 成本（README 已明确警告）。
- 证据：`plugins/explanatory-output-style/README.md:6-7`。

## 风险、边界与改进建议

风险：
1. 校验链路错配风险
- `validate-hook-schema.sh` 与插件 wrapper 格式存在结构性错配，导致“配置可运行但校验失败”的假阳性错误。

2. token 成本与响应冗长风险
- 每次 SessionStart 都注入长 `additionalContext`，在短任务场景有额外成本和输出噪声。

3. 文案漂移风险
- explanatory 与 learning 插件都维护 SessionStart 注入文案，长期可能出现不一致。

边界：
1. 本文件只负责路由
- 不参与权限决策（如 `permissionDecision`）、不处理 `tool_input`。

2. 本文件不承载业务逻辑
- 逻辑在 `hooks-handlers/session-start.sh`；`hooks.json` 只是声明。

3. 本文件无超时与 matcher 显式声明
- 当前条目未配置 `timeout` 和 `matcher`，行为完全依赖运行时默认策略。

改进建议：
1. 修复校验脚本兼容性
- 让 `validate-hook-schema.sh` 自动识别并下钻 `{"hooks": {...}}`，支持插件格式与 settings 格式双栈。

2. 增加轻量回归测试
- 新增针对本插件的 smoke test：校验 `hooks.json` 可解析、handler 输出合法 JSON、`hookEventName` 正确。

3. 明确配置语义
- 评估为 SessionStart 条目显式添加 `matcher: "*"` 和适当 `timeout`（前提是运行时对 SessionStart matcher 的解释一致）。

4. 抽取共享模板
- 将 explanatory 相关 `additionalContext` 文案抽为共享模板，供 `explanatory-output-style` 与 `learning-output-style` 复用，降低漂移。
