# FILE `plugins/learning-output-style/hooks/hooks.json` 研究文档

## 场景与职责

`plugins/learning-output-style/hooks/hooks.json` 是 `learning-output-style` 插件的 Hook 路由入口，职责是把插件能力挂接到 Claude Code 的 `SessionStart` 生命周期事件。

它在链路中的定位是“声明层/路由层”，而不是“执行层/业务层”:
- 上游调用方：Claude Code 插件加载器（通过插件目录发现并读取 hooks 配置）。
- 下游被调用方：`plugins/learning-output-style/hooks-handlers/session-start.sh`（真正输出 `additionalContext` 的执行脚本）。
- 平级依赖：`plugins/learning-output-style/.claude-plugin/plugin.json`（插件元数据，不覆写 hooks 路径）。

关键证据：
- 文件仅声明 `SessionStart -> command` 映射：`plugins/learning-output-style/hooks/hooks.json:1-15`
- 插件默认 hooks 路径是 `./hooks/hooks.json`：`plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:259-263`
- 插件文档定义该能力为“会话启动时注入学习+解释模式上下文”：`plugins/learning-output-style/README.md:3-7,24-27`

## 功能点目的

1. 在会话起点统一注入学习模式行为
- 通过 `SessionStart` 事件确保每次新会话自动生效，而不是依赖用户手动设置。
- 证据：`plugins/learning-output-style/hooks/hooks.json:4`

2. 将学习风格与解释风格合并为一个入口
- 配置只负责触发脚本，脚本输出一段长 `additionalContext`，同时约束“用户贡献 5-10 行关键代码”与“Insight 解释块”。
- 证据：`plugins/learning-output-style/hooks/hooks.json:8-10`，`plugins/learning-output-style/hooks-handlers/session-start.sh:8-11`

3. 保持插件可移植性
- 使用 `${CLAUDE_PLUGIN_ROOT}` 而非硬编码绝对路径，插件可在不同目录安装执行。
- 证据：`plugins/learning-output-style/hooks/hooks.json:9`

4. 与仓库同类 SessionStart 插件保持结构一致
- `explanatory-output-style` 采用同构写法，便于维护者复用认知模型。
- 证据：`plugins/explanatory-output-style/hooks/hooks.json:1-15`

## 具体技术实现（关键流程/数据结构/协议/命令）

### 关键流程

1. 插件被发现
- marketplace 声明 `learning-output-style` 源目录是 `./plugins/learning-output-style`。
- 证据：`.claude-plugin/marketplace.json:95-103`

2. 插件元信息加载
- `plugin.json` 仅声明 name/version/description/author，不覆写 `hooks` 字段。
- 证据：`plugins/learning-output-style/.claude-plugin/plugin.json:1-9`

3. hooks 默认路径解析
- 运行时按默认规则读取 `./hooks/hooks.json`。
- 证据：`plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:259-263`

4. SessionStart 事件触发
- 匹配到 `hooks.SessionStart[0].hooks[0]`，按 `type=command` 执行脚本。
- 证据：`plugins/learning-output-style/hooks/hooks.json:4-10`

5. 脚本返回协议 JSON
- 脚本输出 `hookSpecificOutput.hookEventName=SessionStart` 与 `hookSpecificOutput.additionalContext`，把提示注入会话上下文。
- 证据：`plugins/learning-output-style/hooks-handlers/session-start.sh:8-10`

### 数据结构

目标文件采用“插件 hooks 包装格式”（非 settings 直写格式）：

```json
{
  "description": "Learning mode hook that adds interactive learning instructions",
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

字段语义：
- `description`：配置说明文本。
- `hooks`（顶层）：事件容器。
- `SessionStart`：事件键。
- 事件项对象中的 `hooks[]`：可串接多个 hook；当前仅一个 command hook。
- `command`：真正执行的脚本路径。

格式规范对照：
- 插件格式要求顶层 wrapper：`{"description"?, "hooks": {...}}`。
- 证据：`plugins/plugin-dev/skills/hook-development/SKILL.md:62-80,119`

### 协议与命令

1. 输入协议（命令 Hook）
- Hook 脚本从 stdin 读取 JSON；`SessionStart` 样例包含 `session_id/transcript_path/cwd/permission_mode/hook_event_name`。
- 证据：`plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh:83-92`

2. 输出协议（本插件被调用脚本）
- 返回 `hookSpecificOutput.additionalContext` 来增补会话上下文。
- 证据：`plugins/learning-output-style/hooks-handlers/session-start.sh:8-10`

3. 本次实测命令

- JSON 语法校验：
```bash
jq empty plugins/learning-output-style/hooks/hooks.json
```
结果：通过。

- SessionStart 执行冒烟：
```bash
bash plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh --create-sample SessionStart > /tmp/sample.json
bash plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh plugins/learning-output-style/hooks-handlers/session-start.sh /tmp/sample.json
```
结果：`Exit Code: 0`，输出 JSON 可解析。

- 输出契约检查：
```bash
plugins/learning-output-style/hooks-handlers/session-start.sh | jq -r '.hookSpecificOutput.hookEventName'
plugins/learning-output-style/hooks-handlers/session-start.sh | jq -r '.hookSpecificOutput.additionalContext | length'
```
结果：`hookEventName=SessionStart`，`additionalContext` 长度 `3034` 字符。

- schema 校验脚本行为：
```bash
bash plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh plugins/learning-output-style/hooks/hooks.json
```
结果：先提示 `Unknown event type: description/hooks`，随后 `jq: Cannot index string with number`。

原因分析：
- 该校验脚本按“顶层键即事件键”遍历（`keys[]`），并直接按 `."$event"[$i]` 访问；不兼容插件 wrapper 格式。
- 证据：`plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh:43-70`

## 关键代码路径与文件引用

核心闭环：
1. `plugins/learning-output-style/hooks/hooks.json:1-15`
2. `plugins/learning-output-style/hooks-handlers/session-start.sh:1-15`

调用方与配置来源：
- `.claude-plugin/marketplace.json:95-103`（插件注册）
- `plugins/learning-output-style/.claude-plugin/plugin.json:1-9`（插件元信息）
- `plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:259-263`（hooks 默认路径）

行为文档与产品意图：
- `plugins/learning-output-style/README.md:3-7,11-27,56-82`
- `plugins/README.md:23`（插件目录总览中的定位）

校验与测试脚本（开发上下文依赖）：
- `plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh:83-93,170-191,205-217`
- `plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh:41-75`
- `plugins/plugin-dev/commands/create-plugin.md:201-209,257-260`（官方 workflow 对 hooks 的验证步骤）
- `plugins/plugin-dev/agents/plugin-validator.md:107-114`（插件校验 agent 对 hooks 的要求）

同类对照：
- `plugins/explanatory-output-style/hooks/hooks.json:1-15`
- `plugins/security-guidance/hooks/hooks.json:1-14`
- `plugins/hookify/hooks/hooks.json:1-47`

## 依赖与外部交互

运行时依赖：
- Claude Code Hook 事件系统（`SessionStart`）。
- `bash` 执行环境（command hook）。
- `${CLAUDE_PLUGIN_ROOT}` 环境变量展开。

开发/测试依赖：
- `jq`（JSON 解析/验证）。
- `timeout`（`test-hook.sh` 超时控制）。

外部交互：
- `hooks.json` 本身不访问网络、不直接读写业务代码文件。
- 主要外部交互是与 Claude Hook Runtime 的 stdin/stdout JSON 协议。
- 实际副作用是“每个会话启动都增加系统上下文长度”，README 已警告 token 成本。
- 证据：`plugins/learning-output-style/README.md:7`

## 风险、边界与改进建议

风险：
1. 工具链误报风险
- `validate-hook-schema.sh` 与插件 wrapper 格式不兼容，导致“可运行配置被校验脚本判异常”。

2. token 成本风险
- `SessionStart` 每次注入 3k+ 字符上下文（实测 3034），固定增加上下文负载。

3. 行为漂移风险
- `learning-output-style` 与 `explanatory-output-style` 都维护独立 SessionStart 文案，长期容易语义分叉。

边界：
1. 本文件只做路由，不含业务决策
- 不处理 `tool_input`，不输出 `permissionDecision`，不做阻断策略。

2. 作用域仅会话级
- 当前只挂 `SessionStart`，不覆盖 `PreToolUse/PostToolUse/Stop` 等阶段性治理能力。

3. 当前配置未声明 `matcher/timeout`
- 行为依赖运行时默认值和事件语义。

改进建议：
1. 修复校验脚本兼容性
- 让 `validate-hook-schema.sh` 同时支持：
  - settings 直写格式（事件在顶层）
  - plugin 包装格式（事件在 `.hooks` 下）

2. 增加插件内最小 smoke test
- 在 `plugins/learning-output-style` 增加脚本，至少断言：
  - `hooks.json` 可解析
  - `SessionStart` 路由存在
  - handler 输出包含 `hookSpecificOutput.additionalContext`

3. 评估文案模块化
- 将解释性通用片段抽取为共享模板，降低 `learning`/`explanatory` 双插件维护漂移。

4. 提供轻量模式
- 可考虑短版 `additionalContext`（或通过本地设置开关），在保留学习行为的同时降低 token 消耗。
