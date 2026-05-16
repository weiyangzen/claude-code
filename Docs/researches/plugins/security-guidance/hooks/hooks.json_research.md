# FILE `plugins/security-guidance/hooks/hooks.json` 研究文档

## 场景与职责

`plugins/security-guidance/hooks/hooks.json` 是 `security-guidance` 插件的 Hook 路由配置文件。它的核心职责不是做安全检测本身，而是把 Claude Code 运行时事件精准路由到检测脚本：

- 在 `PreToolUse` 阶段介入（工具执行前）
- 仅匹配写文件相关工具（`Edit|Write|MultiEdit`）
- 以 command hook 方式调用 Python 脚本 `security_reminder_hook.py`

该文件是“声明层”，将运行时事件、匹配规则、执行命令绑定起来；真正的策略逻辑由下游脚本实现。

## 功能点目的

1. 将安全提醒能力前置到工具调用前
- 通过 `PreToolUse` 触发点，在写入真正落盘前给出风险阻断机会。

2. 控制触发范围，避免全局干扰
- `matcher: "Edit|Write|MultiEdit"` 明确限制仅在写文件工具触发，降低对 `Read/Bash` 等工具的性能和行为影响。

3. 保证插件路径可移植
- 命令使用 `${CLAUDE_PLUGIN_ROOT}` 进行路径拼接，避免安装目录变化导致脚本找不到。

4. 与插件系统默认约定对齐
- 本插件 `plugin.json` 未覆写 `hooks` 字段，运行时按默认路径 `./hooks/hooks.json` 加载。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 文件结构与数据模型

该文件采用“插件 wrapper 格式”：

```json
{
  "description": "...",
  "hooks": {
    "PreToolUse": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "python3 ${CLAUDE_PLUGIN_ROOT}/hooks/security_reminder_hook.py"
          }
        ],
        "matcher": "Edit|Write|MultiEdit"
      }
    ]
  }
}
```

关键字段语义：

- `description`：人类可读说明，便于维护。
- `hooks.PreToolUse[]`：事件级配置数组。
- `matcher`：工具名匹配表达式，支持 `|` 组合。
- `hooks[]`：命中后执行的 hook 动作列表。
- `type: command`：执行外部命令。
- `command`：具体执行命令（此处为 Python 脚本）。

### 2) 运行流程

1. Claude Code 启动会话后加载插件 hook 配置。
2. 当即将执行工具时触发 `PreToolUse`。
3. 若工具名匹配 `Edit|Write|MultiEdit`，运行命令：
   - `python3 ${CLAUDE_PLUGIN_ROOT}/hooks/security_reminder_hook.py`
4. 上述脚本读取 stdin JSON，命中风险时以 `exit 2` 阻断，未命中时 `exit 0` 放行。

### 3) 协议与执行语义

- 输入协议：command hook 从 stdin 接收 JSON（含 `session_id`、`tool_name`、`tool_input` 等）。
- 阻断语义：`PreToolUse` 中 `exit 2` 可中断本次工具执行。
- 并行语义：匹配到的多个 hook 可并行执行（本文件当前只配置 1 个 command hook）。

### 4) 校验工具链相关点

`plugin-dev` 提供 `validate-hook-schema.sh` 用于检查 hooks 配置，但该脚本当前按“顶层事件键”遍历 JSON；对于当前文件使用的 wrapper 格式（顶层含 `description/hooks`）会触发 `jq` 索引异常并退出非零。该行为影响自动化校验链路，需要在使用时额外注意。

## 关键代码路径与文件引用

- `plugins/security-guidance/hooks/hooks.json:1-16`（当前文件，事件路由声明）
- `plugins/security-guidance/hooks/security_reminder_hook.py:217-276`（命令下游执行入口与阻断逻辑）
- `plugins/security-guidance/.claude-plugin/plugin.json:1-9`（插件元数据，未覆写 hooks 路径）
- `plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:259-263`（默认 hooks 路径 `./hooks/hooks.json`）
- `plugins/plugin-dev/skills/hook-development/SKILL.md:62-80`（插件 hooks wrapper 格式定义）
- `plugins/plugin-dev/skills/hook-development/SKILL.md:300-317`（hook stdin 输入字段）
- `plugins/plugin-dev/skills/hook-development/SKILL.md:296-299`（`exit 2` 阻断语义）
- `plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh:41-70`（校验脚本当前的顶层遍历假设）
- `plugins/README.md:27`（插件对外能力说明）

## 依赖与外部交互

- 运行依赖：`python3` 可执行环境。
- 运行时注入变量：`CLAUDE_PLUGIN_ROOT`（命令路径拼接）。
- 调用下游：本地脚本 `hooks/security_reminder_hook.py`。
- 输入来源：Claude Code Hook runtime 的 stdin JSON。
- 网络交互：本文件本身无网络访问。
- 测试与脚本：可结合 `plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh` 做端到端回放测试。

## 风险、边界与改进建议

1. 缺少显式 `timeout`
- 现状：command hook 未设置 `timeout`，依赖系统默认值。
- 风险：脚本异常阻塞时，会拖慢工具前置链路。
- 建议：在当前项增加合理超时（例如 10-30 秒）。

2. 匹配范围只按工具名，不按路径/上下文细分
- 现状：所有 `Edit/Write/MultiEdit` 都会触发脚本。
- 风险：大仓库高频写操作下会增加不必要调用。
- 建议：可拆分 matcher + 条件脚本（如仅特定目录启用）。

3. 校验脚本兼容性偏差
- 现状：`validate-hook-schema.sh` 与插件 wrapper 格式存在不兼容。
- 风险：CI/本地自动校验可能误失败，影响开发体验。
- 建议：改造校验脚本支持 `{"hooks": {...}}` 包装层，或先预处理再校验。

4. 单 hook 串接能力有限
- 现状：仅一条 command hook。
- 风险：后续若加入更多前置安全能力，可能缺少层次化治理。
- 建议：按职责拆分为多个 hooks（快速规则、上下文规则、审计日志）并利用并行机制。

